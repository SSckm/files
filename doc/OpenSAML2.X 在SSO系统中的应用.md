##项目介绍
OpenSAML在网上的资料也不是很多，在国内用到的应该也不是很多，自己摸索着做了一下，有可能自己对OpenSAML的理解不够到位

##项目准备

安装Java
安装Maven
安装Eclipse

Maven 配置

配置OpenSAML 依赖
(具体的maven配置可以参考我GitHub上的项目)

```
<dependency>
	<groupId>org.opensaml</groupId>
	<artifactId>opensaml</artifactId>
	<version>2.6.4</version>
</dependency>
```

##使用方式

OpenSAML提供了几种方式来实现SSO之间SAML数据传输。本文主要对HTTP Artifact Binding 做个简单介绍

##项目流程

![](http://img.soaer.com/blog/4.jpg)

1.用户访问SP的受保护资源
2.SP检查是否有用户的Session，如果用则直接访问
3.如果没有Session上下文SP随机生成Artifact，并生成AuthnRequest
如果在Cookie中发现票据信息，把票据信息放到AuthnRequest当中
4.SP建立Artifact与AuthnRequest的关联信息
5.SP重定向到IDP的接受Artifact接口，用Get方式发送Artifact，和SP在IDP中的注册ID
6.IDP接受Artifact，然后用HTTP POST方式来请求SP的getAuthnRequest接口(参数为Artifact)
7.SP 接受到IDP传过来的Artifact ，根据Artifact 把关联的AuthnRequest返回给IDP
8.IDP接受到getAuthnRequest然后来验证AuthnRequest的有效性，检查 Status Version 等信息，如果Cookie中的票据不为空，则检查票据是否正确，是否在有效期内，如果票据为空，则重定向用户到登录页面来提交信息。
9.如果票据正确或者用户通过输入用户名密码等信息通过验证，则IDP生成Artifact对象，IDP生成Response对象，并根据用户信息生成断言，同时对Response 中的 断言做签名处理，对票据对象做加密和签名处理，并把票据信息写入Cookie，并建立Artifact与Response的关联关系，并重定向浏览器到SP的getArtifact接口
10. SP 接受到Artifact，并通过HTTP POST的方式把Artifact发送到IDP
11. IDP通过Artifact找到关联的Response对象返回给SP
    12.SP接受到IDP传输过来的Response对象，首先对Response中的断言做验签操作，如果通过，则同意用户访问资源。

⚠️：其实对于SP和IDP而言重要的信息其实是AuthnRequest和Response，只不过每次传输到浏览器的时候都是传递的是各自的引用，然后SP和IDP再更具饮用来获取 真实的数据。这样做安全性可能更高一点，但是增加了SP和IDP之间的通信。

⚠️：其中用到的证书都是自己用OpenSSL自己制定。
参考地址：http://blog.csdn.net/howeverpf/article/details/21622545


SP端发送Artifact的JSP页面
```
<body>
	<form action="<%=SysConstants.IDPRECEIVESPARTIFACT_URL%>" method="get" name = "autoForm">
		<input type="hidden" name="artifact" value="<%=request.getAttribute(SysConstants.ARTIFACT_KEY)%>" />
		<input type="hidden" name="token" value="<%=request.getAttribute(SysConstants.TOKEN_KEY)%>" />
	</form>
</body>
<script type="text/javascript">
	document.autoForm.submit();
</script>
```

IDP 端接收Artifact的代码
```
    // 获取Artifact
    String artifactBase64 = request.getParameter(SysConstants.ARTIFACT_KEY);
    if (null == artifactBase64) {
      throw new RuntimeException("SP端传递的Artifact参数错误，该参数不可为空");
    }
    final ArtifactResolve artifactResolve = samlService.buildArtifactResolve();
    final Artifact artifact = (Artifact) samlService.buildStringToXMLObject(artifactBase64);
    artifactResolve.setArtifact(artifact);
    // 对artifactResolve对象进行签名操作
    samlService.signXMLObject(artifactResolve);
    String artifactParam = samlService.buildXMLObjectToString(artifactResolve);
    String postResult = null;
    //根据Artifact信息从SP中获取Auth Request信息
    try {
      postResult = HttpUtil.doPost(SysConstants.SP_ARTIFACT_RESOLUTION_SERVICE, artifactParam);
    } catch (Exception e) {
      throw new RuntimeException("访问SP端的：" + SysConstants.SP_ARTIFACT_RESOLUTION_SERVICE + "服务错误" + e.getMessage());
    }
    if (null == postResult || "".equals(postResult)) {
      return "redirect:/loginPage";
    }
    final ArtifactResponse artifactResponse = (ArtifactResponse) samlService.buildStringToXMLObject(postResult);
    final Status status = artifactResponse.getStatus();
    if (null == status) {
      throw new RuntimeException("客户端的Status不能为空");
    }
    StatusCode statusCode = status.getStatusCode();
    if (null == statusCode) {
      throw new RuntimeException("无法获取statusCode");
    }
    String codeValue = statusCode.getValue();
    // 判断SAML的StatusCode
    if (codeValue == null || !codeValue.equals(StatusCode.SUCCESS_URI)) {
      throw new RuntimeException("无法获取codeValue， 认证失败， SP端数据错误");
    }
    final String inResponseTo = artifactResponse.getInResponseTo();
    final String artifactResolveID = artifactResolve.getID();
    if (null == inResponseTo || !inResponseTo.equals(artifactResolveID)) {
      logger.error("artifact的值和接收端的inResponseTo的值不一致，认证错误");
      throw new RuntimeException("artifact的值和接收端的inResponseTo的值不一致，认证错误");
    }
    final AuthnRequest authnRequest = (AuthnRequest) artifactResponse.getMessage();
    if (authnRequest == null) {
      throw new RuntimeException("无法获取AuthRequest数据，认证错误");
    }
    // 获取SP的消费URL，下一步回调需要用到
    final String customerServiceUrl = authnRequest.getAssertionConsumerServiceURL();
    if (null == customerServiceUrl) {
      throw new RuntimeException("无法获取customerServiceUrl，SP端数据错误");
    }
    request.setAttribute(SysConstants.ACTION_KEY, customerServiceUrl);
    HttpSession session = request.getSession(false);
    session.setAttribute(SysConstants.ACTION_KEY, customerServiceUrl);
    final SAMLVersion samlVersion = authnRequest.getVersion();

    // 判断版本是否支持
    if (null == samlVersion || !SAMLVersion.VERSION_20.equals(samlVersion)) {
      throw new RuntimeException("SAML版本错误，只支持2.0");
    }

    final Issuer issuer = authnRequest.getIssuer();
    final String appName = issuer.getValue();

    // 判断issure的里面的值是否在SSO系统中注册过
    final App app = appService.findAppByAppName(appName.trim());
    if (app == null) {
      throw new RuntimeException("不支持当前系统: " + appName);
    }
    
    Subject subject = authnRequest.getSubject();
    NameID nameID = subject.getNameID();
    String ticket = nameID.getValue();
    if ("".equals(ticket) || null == ticket) {
      return "redirect:/loginPage";
    }
    try {
      ticket = URLDecoder.decode(ticket, SysConstants.CHARSET);
    } catch (UnsupportedEncodingException e) {
      e.printStackTrace();
    }
    /**
     * 解密ticket
     */
    PrivateKey privateKey = samlService.getRSAPrivateKey();
    String[] afterDecode = null;
    try {
      byte[] ticketArray = Base64.decode(ticket);
      byte[] decryptTicketArray = RSACoder.INSTANCE.decryptByPrivateKey(privateKey, ticketArray);
      String decryptTicket = new String(decryptTicketArray);
      afterDecode = decryptTicket.split(SysConstants.TICKET_SPILT);
    } catch (Exception e) {
      return "redirect:/loginPage";
    }
    logger.debug("Ticket验证痛过");
    // 判断令牌是否过期，如果令牌过期则直接
    if (afterDecode == null || afterDecode.length != 3) {
      return "redirect:/loginPage";
    }
    long expireTime = Long.parseLong(afterDecode[2]);
    long nowTime = System.currentTimeMillis();
    if (expireTime < nowTime) {
      logger.debug("Token已过期");
      return "redirect:/loginPage";
    }
    final Artifact idpArtifact = samlService.buildArtifact();
    final Response samlResponse = samlService.buildResponse(UUIDFactory.INSTANCE.getUUID());
    long id = Long.parseLong(afterDecode[0]);
    User user = new User();
    user.setId(id);
    user.setEmail(afterDecode[1]);
    samlService.addAttribute(samlResponse, user);
    SSOHelper.INSTANCE.put(idpArtifact.getArtifact(), samlResponse);
    request.setAttribute(SysConstants.ARTIFACT_KEY, samlService.buildXMLObjectToString(idpArtifact));
    return "/saml/idp/send_artifact_to_sp";

```

SP接受Artifact的代码

```
    String spArtifact = request.getParameter(SysConstants.ARTIFACT_KEY);
    if (null == spArtifact) {
      throw new RuntimeException("无法获取IDP端传过来的Artifact");
    }
    ArtifactResolve artifactResolve = samlService.buildArtifactResolve();
    Artifact artifact = (Artifact) samlService.buildStringToXMLObject(spArtifact);
    artifactResolve.setArtifact(artifact);
    samlService.signXMLObject(artifactResolve);
    String requestStr = samlService.buildXMLObjectToString(artifactResolve);
    String postResult = null;
    try {
      postResult = HttpUtil.doPost(SysConstants.IDP_ARTIFACT_RESOLUTION_SERVICE, requestStr);
    } catch (Exception e) {
      e.printStackTrace();
      logger.error("访问IDP的" + SysConstants.IDP_ARTIFACT_RESOLUTION_SERVICE + "服务错误");
    }
    if (null == postResult) {
      throw new RuntimeException("从" + SysConstants.IDP_ARTIFACT_RESOLUTION_SERVICE + "服务获取的数据为空，请检查IDP端数据格式");
    }
    ArtifactResponse artifactResponse = (ArtifactResponse) samlService.buildStringToXMLObject(postResult);
    SAMLObject samlObject = artifactResponse.getMessage();
    if (null == samlObject) {
      throw new RuntimeException("无法获取Response信息");
    }
    Response samlResponse = (Response) samlObject;
    List<Assertion> assertions = samlResponse.getAssertions();
    if (null == assertions || assertions.size() == 0) {
      throw new RuntimeException("无法获取断言，请重新发起请求！！！");
    }
    Assertion assertion = samlResponse.getAssertions().get(0);
    if (assertion == null) {
      request.setAttribute(SysConstants.ERROR_LOGIN, true);
    } else {
      /**
       * 验证签名
       */
      boolean signSuccess = samlService.validate(assertion);
      if (!signSuccess) {
        throw new RuntimeException("验证签名错误");
      }
      HttpSession session = request.getSession(false);
      List<AttributeStatement> arrtibuteStatements = assertion.getAttributeStatements();
      if (null == arrtibuteStatements || arrtibuteStatements.size() == 0) {
        throw new RuntimeException("无法获取属性列表，请重新发起请求");
      }
      AttributeStatement attributeStatement = assertion.getAttributeStatements().get(0);
      List<Attribute> list = attributeStatement.getAttributes();
      if (null == list) {
        throw new RuntimeException("无法获取属性列表IDP端错误");
      }
      User user = new User();
      list.forEach(pereAttribute -> {
        String name = pereAttribute.getName();
        XSString value = (XSString) pereAttribute.getAttributeValues().get(0);
        String valueString = value.getValue();
        if (name.endsWith("Name")) {
          user.setName(valueString);
        } else if (name.equals("Id")) {
          user.setId(Long.parseLong(valueString));
        } else if (name.equals("LoginId")) {
          user.setLogin_id(valueString);
        } else if (name.equals("Email")) {
          user.setEmail(valueString);
        }
      });
      session.setAttribute(SysConstants.LOGIN_USER, user);
      putAuthnToSecuritySession("admin", "admin");
      request.setAttribute(SysConstants.ERROR_LOGIN, false);
    }
```

