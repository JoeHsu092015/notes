---
layout: post
title: Logstash with input s3 plugin
date: 2020-04-18 17:32:20 +0800
description: fetch data from amazon S3 service # Add post description (optional)
#img: i-rest.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [elk, aws]
categories: infrastructure
---

在使用 s3 plugin 時候, 遇到 java error message 時候, 在 log file 會印出多行的 error stack trace. 可以使用 codec **multiline** 來解決, 把這些 error message 合併成一個 log message.

**問題**: 
使用 multiline  後, add_field 和 @metadata 的資料會在每個 log file 的最後一筆 message 被清空, 導致讀不到設定的值. 

github 有人提出, 但是目前還沒把解決方法更新.
https://github.com/logstash-plugins/logstash-input-s3/pull/173


**解決**:
自己安裝 ruby 使用 gem 從 source code 自己加入網友提供的程式碼 build 一版新的 `logstash-input-s3-3.5.1.gem`

在 source code 的 lib/logstash/inputs/s3.rb 加入:

```ruby=
# #ensure any stateful codecs (such as multi-line ) are flushed to the queue
    @codec.flush do |event|
      #-------new-----------
      decorate(event)
      
      event.set("cloudfront_version", metadata[:cloudfront_version]) unless metadata[:cloudfront_version].nil?
      event.set("cloudfront_fields", metadata[:cloudfront_fields]) unless metadata[:cloudfront_fields].nil?

      if @include_object_properties
        event.set("[@metadata][s3]", object.data.to_h)
      else
        event.set("[@metadata][s3]", {})
      end

      event.set("[@metadata][s3][key]", object.key)
      #-------new end-----------
      queue << event
    end
```


並在 Dockerfile 裡面安裝
Dockerfile:

```dockerfile=
FROM docker.elastic.co/logstash/logstash-oss:7.4.2
RUN rm -f /usr/share/logstash/pipeline/logstash.conf

COPY logstash-input-s3-3.5.1.gem /
RUN /usr/share/logstash/bin/logstash-plugin install /logstash-input-s3-3.5.1.gem && \
 /usr/share/logstash/bin/logstash-plugin install logstash-filter-json
ADD pipeline/ /usr/share/logstash/pipeline/
ADD config/ /usr/share/logstash/config/
```


* **java tomcat errorlog**

```java=
2016-04-07 15:27:28,459 [http-bio-8080-exec-33] ERROR v1.PaymentTxController  - Cannot get property 'attrs' on null object
java.lang.NullPointerException: Cannot get property 'attrs' on null object
    at com.b2boost.payment.provider.paybox.PayboxPaymentProviderService.createSubscriptionAndPay(PayboxPaymentProviderService.groovy:206)
    at com.b2boost.payment.provider.paybox.PayboxPaymentProviderService$__tt__pay_closure9.doCall(PayboxPaymentProviderService.groovy:82)
    at com.b2boost.commons.error.AppError.safe(AppError.groovy:53)
    at com.b2boost.commons.error.AppError.safe(AppError.groovy:60)
    at com.b2boost.payment.provider.paybox.PayboxPaymentProviderService.$tt__pay(PayboxPaymentProviderService.groovy:73)
    at com.b2boost.payment.PaymentService$__tt__pay_closure8.doCall(PaymentService.groovy:52)
    at com.b2boost.commons.error.AppError.safeWithEither(AppError.groovy:70)
    at com.b2boost.commons.error.AppError.safeWithEither(AppError.groovy:64)
    at com.b2boost.payment.PaymentService.$tt__pay(PaymentService.groovy:43)
    at com.b2boost.users.api.v1.PaymentTxController$_save_closure1.doCall(PaymentTxController.groovy:49)
    at com.b2boost.users.api.v1.BaseController.documentWithAuthorization(BaseController.groovy:101)
    at com.b2boost.users.api.v1.PaymentTxController.save(PaymentTxController.groovy:45)
    at grails.plugin.cache.web.filter.PageFragmentCachingFilter.doFilter(PageFragmentCachingFilter.java:177)
    at grails.plugin.cache.web.filter.AbstractFilter.doFilter(AbstractFilter.java:63)
    at com.odobo.grails.plugin.springsecurity.rest.RestTokenValidationFilter.processFilterChain(RestTokenValidationFilter.groovy:99)
    at com.odobo.grails.plugin.springsecurity.rest.RestTokenValidationFilter.doFilter(RestTokenValidationFilter.groovy:66)
    at grails.plugin.springsecurity.web.filter.GrailsAnonymousAuthenticationFilter.doFilter(GrailsAnonymousAuthenticationFilter.java:53)
    at com.odobo.grails.plugin.springsecurity.rest.RestAuthenticationFilter.doFilter(RestAuthenticationFilter.groovy:108)
    at grails.plugin.springsecurity.web.authentication.logout.MutableLogoutFilter.doFilter(MutableLogoutFilter.java:82)
    at com.odobo.grails.plugin.springsecurity.rest.RestLogoutFilter.doFilter(RestLogoutFilter.groovy:63)
    at com.brandseye.cors.CorsFilter.doFilter(CorsFilter.java:82)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
    at java.lang.Thread.run(Thread.java:745)
```


* **logstash.conf**

```json=
input {
  s3 {
    access_key_id => "sssssssss"
    secret_access_key => "vvvvvvvvvvv"
    bucket => "my-bucket"
    region => "us-east-1"
    prefix => "logs/"
    additional_settings => {
      force_path_style => true
      follow_redirects => false
    }

    codec => multiline {
      pattern => "^%{TIMESTAMP_ISO8601} "
      negate => true
      what => "previous"
    }

    add_field => {
      "service_name"=>"my-service"
    }
  }
 
}

filter {
  
  grok {
    break_on_match => true
    match => {
      "message" => [
        # java log pattern
        "^%{TIMESTAMP_ISO8601:timestamp} \[%{GREEDYDATA:process_id}\] %{LOGLEVEL:message_level} %{GREEDYDATA:service_message}$",
        "^%{TIMESTAMP_ISO8601:timestamp} %{GREEDYDATA:service_message}$"
      ]
    }
  }

  date {
    match =>["timestamp","yyyy-MM-dd'T'HH:mm:ssZ","yyyy/MM/dd - HH:mm:ss","ISO8601"]
    target =>"@timestamp"
    locale=>"en"
  }

  mutate { 
    add_field => { "file_name" => "%{[@metadata][s3][key]}"}
  }

}


output {
  elasticsearch {
      hosts => [ "elasticsearch:9200" ]
      index => "%{service_name}-%{+YYYY.MM.dd}"
  }
}
```

