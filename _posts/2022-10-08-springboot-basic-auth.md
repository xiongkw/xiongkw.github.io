---
layout: post
title: SpringBoot的Basic Auth实现
categories: [java, springboot]
tags: []
---

> 

#### 1. 编写Filter
```
@Configuration
public class AuthConfig {
    @Value("${auth.username:admin}")
    private String username;
    @Value("${auth.password:123456")
    private String password;

    @Bean
    public FilterRegistrationBean<Filter> filterRegistration() {
        FilterRegistrationBean<Filter> bean = new FilterRegistrationBean<>();
        bean.setFilter((req, res, chain) -> {
            String header = ((HttpServletRequest) req).getHeader("Authorization");
            if (header != null && header.length() > 6) {
                String auth = header.substring(0, 5);
                if ("basic".equalsIgnoreCase(auth)) {
                    String base64 = header.substring(6);
                    String str = new String(Base64.decodeBase64(base64));
                    int index = str.indexOf(":");
                    if (index > 0) {
                        String[] split = str.split(":");
                        if (this.username.equals(split[0]) && this.password.equals(split[1])) {
                            chain.doFilter(req, res);
                            return;
                        }
                    }
                }
            }
            log.info("Auth failed, auth info: [{}]", header);
            ((HttpServletResponse) res).setHeader("WWW-Authenticate", "Basic realm=\"Secure Area\"");
            ((HttpServletResponse) res).setStatus(HttpStatus.UNAUTHORIZED.value());
        });
        bean.addUrlPatterns("/*");
        bean.setName("AuthFilter");
        bean.setOrder(0);
        return bean;
    }
}
```
