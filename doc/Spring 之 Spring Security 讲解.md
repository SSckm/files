

#准备工作

安装Java（不做说明）
安装Maven
http://blog.csdn.net/mrsunnycream/article/details/45025135

#Maven 配置

```
<dependency>
	<groupId>org.springframework.security</groupId>
	<artifactId>spring-security-web</artifactId>
	<version>4.0.4.RELEASE</version>
</dependency>

<dependency>
	<groupId>org.springframework.security</groupId>
	<artifactId>spring-security-config</artifactId>
	<version>4.0.4.RELEASE</version>
</dependency>

<dependency>
	<groupId>org.springframework.security</groupId>
	<artifactId>spring-security-taglibs</artifactId>
	<version>4.0.4.RELEASE</version>
</dependency>
```

#Spring  Security Filter 列表

![](http://img.soaer.com/blog/3.jpg)

经常用到的Filter列表

```
 1. CONCURRENT_SESSION_FILTER
 2. CSRF_FILTER
 3. LOGOUT_FILTER
 4. CAS_FILTER
 5. FORM_LOGIN_FILTER
 6. SESSION_MANAGEMENT_FILTER
 7. FILTER_SECURITY_INTERCEPTOR
```

#Filter 说明

###CONCURRENT_SESSION_FILTER
这个Filter顾名思义就是控制Session并发的，比如同一个用户在不同的地方登录，那么先登录的再次刷新的时候就会是无效登录。

###CSRF_FILTER
这个Filter 主要是针对CSRF攻击的，Security 4.x版本后会默认开启这个Filter

如果要禁用这个功能
使用下面的配置
```
<http use-expressions="true" entry-point-ref="authenticationEntryPoint" >
		<csrf disabled="true" />
</http>
```
###LOGOUT_FILTER

主要是做安全退出使用的（包括销毁Spring Security 的Session）

###CAS_FILTER
这个Filter 主要是集成SSO服务用的Filter，如果使用CAS_FILTER，和Spring Security的权限控制，就可以在这个Filter中来生成用户的登录凭证到 Spring Security Session当中去。

###FORM_LOGIN_FILTER
这个Filter主要是操作登录用到的，进行登录验证

###SESSION_MANAGEMENT_FILTER
这个Filter功能很强大，主要是防止Session固定攻击的，但是可以配置，有三个配置，第一个是使用原有的Session，第二个是复制一个Session 但是不复制原来Session的属性，第三个是完全复制Session。
不知道的同学可以Google一下 Session 固化

###FILTER_SECURITY_INTERCEPTOR

这个Filter主要是做权限提供和验证用户是否有访问某一个资源的权限

注：本文主要是学习

FORM_LOGIN_FILTER
LOGOUT_FILTER
FILTER_SECURITY_INTERCEPTOR

CONCURRENT_SESSION_FILTER可以查看官网的配置也很简单。
CAS_FILTER 我现在用的是OpenSAML实现的认证中心的方式，信息交互不很一样所以没有用到CAS_FILTER

CSRF_FILTER现在我是禁用的


#配置web.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="3.0" xmlns="http://java.sun.com/xml/ns/javaee"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://java.sun.com/xml/ns/javaee 
    http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd">
	<context-param>
		<param-name>webAppRootKey</param-name>
		<param-value>org.sms.root</param-value>
	</context-param>

	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>/WEB-INF/config/spring/*.xml</param-value>
	</context-param>
	<filter>
		<filter-name>encodingFilter</filter-name>
		<filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
		<init-param>
			<param-name>encoding</param-name>
			<param-value>UTF-8</param-value>
		</init-param>
		<init-param>
			<param-name>forceEncoding</param-name>
			<param-value>true</param-value>
		</init-param>
	</filter>
	<filter-mapping>
		<filter-name>encodingFilter</filter-name>
		<url-pattern>/*</url-pattern>
	</filter-mapping>
	<filter>
		<filter-name>SamlFilter</filter-name>
		<filter-class>org.sms.project.filter.SamlFilter</filter-class>
	</filter>
	<filter-mapping>
		<filter-name>SamlFilter</filter-name>
		<url-pattern>/*</url-pattern>
	</filter-mapping>
	<context-param>
		<param-name>log4jConfigLocation</param-name>
		<param-value>/WEB-INF/config/log4j/log4j.xml</param-value>
	</context-param>
	<context-param>
		<param-name>log4jRefreshInterval</param-name>
		<param-value>60000</param-value>
	</context-param>

	<filter>
		<filter-name>springSecurityFilterChain</filter-name>
		<filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
	</filter>
	<filter-mapping>
		<filter-name>springSecurityFilterChain</filter-name>
		<url-pattern>/*</url-pattern>
	</filter-mapping>

	<listener>
		<listener-class>org.springframework.web.util.Log4jConfigListener</listener-class>
	</listener>
	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>
	<listener>
		<listener-class>org.sms.project.listener.SysContextLoaderListener</listener-class>
	</listener>
	<listener>
		<listener-class>org.sms.project.listener.SysSessionListener</listener-class>
	</listener>
	<listener>
		<listener-class>org.springframework.security.web.session.HttpSessionEventPublisher</listener-class>
	</listener>
	<servlet>
		<servlet-name>springServlet</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>/WEB-INF/config/dispatcher-servlet.xml</param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
	</servlet>
	<servlet-mapping>
		<servlet-name>springServlet</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>

	<session-config>
		<session-timeout>60</session-timeout>
	</session-config>
	<welcome-file-list>
		<welcome-file>index.html</welcome-file>
	</welcome-file-list>
	<error-page>
		<error-code>404</error-code>
		<location>/jsp/error/404.jsp</location>
	</error-page>
	<!-- 500错误拦截 -->
	<error-page>
		<error-code>500</error-code>
		<location>/jsp/error/500.jsp</location>
	</error-page>
</web-app>
```

##Spring Security功能详解

###LoginFilter 定义

applicationContext-security.xml 配置说明

```
<beans:bean id="loginFilter" class="org.sms.project.login.LoginFilter">
		<beans:property name="filterProcessesUrl" value="/loginFilter"></beans:property>
		<beans:property name="authenticationManager" ref="authenticationManager"></beans:property>
		<beans:property name="usernameParameter" value="username" />
		<beans:property name="passwordParameter" value="password" />
		<beans:property name="authenticationSuccessHandler" ref="loginSuccessHandler"></beans:property>
		<beans:property name="authenticationFailureHandler" ref="loginFailureHandler"></beans:property>
</beans:bean>
```

注：上面的配置是自定义LoginFilter,也可以使用Security自带的Filter

LoginFilter.java
```
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import org.sms.SysConstants;
import org.sms.organization.user.entity.User;
import org.sms.organization.user.service.UserService;
import org.sms.project.encrypt.md5.MD5;
import org.sms.project.helper.SessionHelper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.AuthenticationServiceException;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

/**
 * 用户登录Filter
 * @author Sunny
 */
public class LoginFilter extends UsernamePasswordAuthenticationFilter {

  @Autowired
  private UserService userService;

  @Override
  public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
    
    /**
     * 登录方法判断，仅支持POST提交
     */
    if (!SysConstants.POST_METHOE.equals(request.getMethod())) {
      throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
    }
    
    String username = obtainUsername(request);
    String password = obtainPassword(request);
    if (null == username || null == password || username.equals(""))
      throw new AuthenticationServiceException("用户名或者密码为空！");
    username = username.trim();
    User user = this.userService.findUserByLoginId(username);
    if (null == user)
      throw new AuthenticationServiceException("用户不存在！");
    if (!user.isEnabled())
      throw new AuthenticationServiceException("用户已经被锁定！");
    if (!password.equals(user.getPassword()))
      throw new AuthenticationServiceException("用户名或者密码错误！");
    UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(username, salt);
    setDetails(request, authRequest);
    SessionHelper.put(request, SysConstants.LOGIN_USER, user);
    return this.getAuthenticationManager().authenticate(authRequest);
  }
}
```
当然如果使用的SSO登录的统一认证中心的话就省略了这个
LoginFilter

### FILTER_SECURITY_INTERCEPTOR
applicationContext-security.xml

```

<http use-expressions="true" entry-point-ref="authenticationEntryPoint" >
		<csrf disabled="true" />
		<custom-filter ref="loginFilter" position="FORM_LOGIN_FILTER" />
		<custom-filter ref="customLogoutFilter" position="LOGOUT_FILTER" />
		<custom-filter ref="securityFilter" before="FILTER_SECURITY_INTERCEPTOR" />
</http>
<beans:bean id="authenticationEntryPoint" class="org.springframework.security.web.authentication.LoginUrlAuthenticationEntryPoint">
		<beans:constructor-arg value="/html/authen.html" />
</beans:bean>
<beans:bean id="securityFilter" class="org.sms.project.filter.SecurityFilter">
		<beans:property name="accessDecisionManager" ref="accessDecisionManager"></beans:property>
		<beans:property name="securityMetadataSource" ref="securityMetadataSource"></beans:property>
		<beans:property name="authenticationManager" ref="authenticationManager"></beans:property>
	</beans:bean>

	<beans:bean id="securityMetadataSource" class="org.sms.project.security.SysMetadataSource">
		<beans:property name="resourceService" ref="resourceService"></beans:property>
	</beans:bean>

	<beans:bean id="accessDecisionManager" class="org.sms.project.security.SysDecisionManager">
	</beans:bean>

	<authentication-manager>
		<authentication-provider user-service-ref="sysUserDetailsService"></authentication-provider>
	</authentication-manager>
	<beans:bean id="sysUserDetailsService" class="org.sms.project.security.SysUserDetailsService">
		<authentication-manager alias="authenticationManager">
		<authentication-provider user-service-ref="sysUserDetailsService"></authentication-provider>
	</authentication-manager>
</beans:bean>

```

SecurityFilter.java
```
import java.io.IOException;
import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import org.springframework.security.access.SecurityMetadataSource;
import org.springframework.security.access.intercept.AbstractSecurityInterceptor;
import org.springframework.security.access.intercept.InterceptorStatusToken;
import org.springframework.security.web.FilterInvocation;
import org.springframework.security.web.access.intercept.FilterInvocationSecurityMetadataSource;

/**
 * @author Sunny
 */
public class SecurityFilter extends AbstractSecurityInterceptor implements Filter {
  
  private FilterInvocationSecurityMetadataSource securityMetadataSource;

  @Override
  public SecurityMetadataSource obtainSecurityMetadataSource() {
    return this.securityMetadataSource;
  }

  public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
    FilterInvocation fi = new FilterInvocation(request, response, chain);
    invoke(fi);
  }

  private void invoke(FilterInvocation fi) throws IOException, ServletException {
    InterceptorStatusToken token = super.beforeInvocation(fi);
    try {
      fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
    } finally {
      super.afterInvocation(token, null);
    }
  }

  public FilterInvocationSecurityMetadataSource getSecurityMetadataSource() {
    return securityMetadataSource;
  }

  public void setSecurityMetadataSource(FilterInvocationSecurityMetadataSource securityMetadataSource) {
    this.securityMetadataSource = securityMetadataSource;
  }

  public void init(FilterConfig arg0) throws ServletException {
  }

  public void destroy() {
  }

  @Override
  public Class<? extends Object> getSecureObjectClass() {
    return FilterInvocation.class;
  }
}
```

SysDecisionManager.java
```
import java.util.Collection;
import org.springframework.security.access.AccessDecisionManager;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.access.ConfigAttribute;
import org.springframework.security.authentication.InsufficientAuthenticationException;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;

/**
 * @author Sunny
 */
public class SysDecisionManager implements AccessDecisionManager {

  @Override
  public void decide(Authentication authentication, Object object, Collection<ConfigAttribute> configAttributes) throws AccessDeniedException, InsufficientAuthenticationException {
    for (ConfigAttribute allConfigAttribute : configAttributes) {
      final String needPermission = allConfigAttribute.getAttribute();
      for (GrantedAuthority ga : authentication.getAuthorities()) {
        if (needPermission.equals(ga.getAuthority())) {
          return;
        }
      }
    }
    throw new AccessDeniedException("您访问的资源没有权限！！");
  }

  @Override
  public boolean supports(ConfigAttribute configAttribute) {
    return true;
  }

  @Override
  public boolean supports(Class<?> clazz) {
    return true;
  }
}
```


SysMetadataSource.java
```
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import javax.servlet.http.HttpServletRequest;
import org.sms.project.resource.dao.ResourceDao.ResourceMapping;
import org.sms.project.resource.service.ResourceService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.access.ConfigAttribute;
import org.springframework.security.access.SecurityConfig;
import org.springframework.security.web.FilterInvocation;
import org.springframework.stereotype.Service;
import org.springframework.security.web.util.matcher.AntPathRequestMatcher;
import org.springframework.security.web.util.matcher.RequestMatcher;
import org.springframework.security.web.access.intercept.FilterInvocationSecurityMetadataSource;

/**
 * 资源角色管理器
 * @author Sunny
 */
@Service("sysMetadataSource")
public class SysMetadataSource implements FilterInvocationSecurityMetadataSource {

  @Autowired
  private ResourceService resourceService;

  private Map<String, List<ConfigAttribute>> configAttributes;

  private RequestMatcher requestMatcher = null;

  public SysMetadataSource() {
    super();
  }

  public final ResourceService getResourceService() {
    return resourceService;
  }

  public final void setResourceService(ResourceService resourceService) {
    this.resourceService = resourceService;
  }

  @Override
  public Collection<ConfigAttribute> getAttributes(Object object) throws IllegalArgumentException {

    final HttpServletRequest request = ((FilterInvocation) object).getRequest();
    if (null == configAttributes)
      this.load();
    final List<ConfigAttribute> list = new ArrayList<ConfigAttribute>();
    configAttributes.forEach((K, V) -> {
      requestMatcher = new AntPathRequestMatcher(K);
      if (null != K) {
        if (requestMatcher.matches(request)) {
          if (null != V) {
            list.addAll(V);
          }
        }
      }
    });
    return list;
  }

  /**
   * 加载所有的url访问时需要的角色
   */
  private void load() {
    List<ResourceMapping> resourceMappings = resourceService.getResourceMappings();
    if (resourceMappings.size() == 0)
      return;
    this.configAttributes = new HashMap<String, List<ConfigAttribute>>();
    resourceMappings.forEach(resourceMapping -> {
      final List<ConfigAttribute> list = new ArrayList<ConfigAttribute>(1);
      final String url = resourceMapping.getUrl();
      final String role_name = resourceMapping.getRole_name();
      if (null != url) {
        ConfigAttribute configAttribute = new SecurityConfig(role_name);
        list.add(configAttribute);
        final List<ConfigAttribute> isExit = configAttributes.get(url);
        if (isExit == null) {
          configAttributes.put(url, list);
        } else {
          if (isExit.contains(configAttribute)) {
            configAttributes.put(url, list);
          } else {
            isExit.addAll(list);
            configAttributes.put(url, isExit);
          }
        }
      }
    });
  }
  
  public void addResourceToConfigAttributes(String url, String ...role_names) {
    if (null == url || null == role_names || role_names.length == 0)
      return;
    ConfigAttribute configAttribute = null;
    List<ConfigAttribute> list = new ArrayList<ConfigAttribute>(role_names.length);
    if (configAttributes.get(url) == null) {
      for (String role_name : role_names) {
        configAttribute = new SecurityConfig(role_name);
        list.add(configAttribute);
      }
      configAttributes.put(url, list);
    } else {
      final List<ConfigAttribute> isExit = configAttributes.get(url);
      for (String role_name : role_names) {
        configAttribute = new SecurityConfig(role_name);
        if (!isExit.contains(configAttribute)) {
          isExit.add(configAttribute);
        }
      }
      configAttributes.put(url, isExit);
    }
  }

  @Override
  public boolean supports(Class<?> clazz) {
    return true;
  }

  @Override
  public Collection<ConfigAttribute> getAllConfigAttributes() {
    return null;
  }
}
```

SysUserDetailsService.java
```
import java.util.HashSet;
import java.util.Set;
import org.sms.organization.user.entity.User;
import org.sms.organization.user.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;

/**
 * @author Sunny
 */
public class SysUserDetailsService implements UserDetailsService {

  @Autowired
  private UserService sysUserService;

  @Override
  public UserDetails loadUserByUsername(String login_id) throws UsernameNotFoundException {
    User user = sysUserService.findUserByLogin_Id(login_id);
    this.buildAuths(user);
    return user;
  }

  private void buildAuths(User user) {
    Set<GrantedAuthority> auths = new HashSet<GrantedAuthority>();
    auths.add(new SimpleGrantedAuthority("ADMIN"));
    user.setAuthorities(auths);
  }
}
```

##当使用Spring  Security 自定义增加凭证到Session
在不使用CASFilter的情况下

SampleAuthenticationManager.java
```
import java.util.ArrayList;
import java.util.List;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;

/**
 * @author Sunny
 */
public class SampleAuthenticationManager implements AuthenticationManager {

  private List<String> list;

  public SampleAuthenticationManager(List<String> list) {
    this.list = list;
  }

  @Override
  public Authentication authenticate(Authentication authentication) throws AuthenticationException {
    final List<GrantedAuthority> authoritys = new ArrayList<GrantedAuthority>();
    list.forEach(pereRoleName -> {
      authoritys.add(new SimpleGrantedAuthority(pereRoleName));
    });
    return new UsernamePasswordAuthenticationToken(authentication.getName(), authentication.getCredentials(), authoritys);
  }
}
```

调用SampleAuthenticationManager.java
```
	final List<String> list = new ArrayList<String>();
    list.add("ADMIN");
    AuthenticationManager authenticationManager = new SampleAuthenticationManager(list);
    //其中password可以是票据之类的对象
    Authentication request = new UsernamePasswordAuthenticationToken(name, password);
    Authentication result = authenticationManager.authenticate(request);
    SecurityContextHolder.getContext().setAuthentication(result);
```




