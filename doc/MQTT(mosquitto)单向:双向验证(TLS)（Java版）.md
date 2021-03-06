# MQTT(mosquitto)单向/双向验证(TLS)（Java版）



> 本篇文章主要讲解MQTT单向和双向验证Java实现,在实现过程中也走了很多弯路，所以记录下来供以后查阅。
>
> ​	1.本篇文章使用的是公共的Broker,其中CA证书也是从test.mosquitto.org下载的，所以单向认证只需要mosquitto.org.crt一个CA证书就可以了。双向验证需要自己生成证书请求文件然后让后在http://test.mosquitto.org/ssl/上申请签名，就可以自动下载了。
>
> ​	2.如果自己生成证书，那么需要生成ca证书，服务器证书，客户端证书，三种证书（可以用OpenSSL生成）也比较简单。

##下面介绍两种方式

###使用公共Broker

```
地址：http://test.mosquitto.org/
其中：1883不加密信道
	 8883单向加密信道
	 8884双向加密信道
	 其中还有WebSocket端口，本篇文章主要介绍tpc/ssl(tls)
```

###自己安装MQTT Broker

####Mac安装

```
brew install mosquitto
```

####Ubuntu安装

```
sudo apt-add-repository ppa:mosquitto-dev/mosquitto-ppa
sudo apt-get update
sudo apt-get install mosquitto
sudo apt-get install mosquitto-clients
```

####修改配置文件

```
pid_file /var/run/mosquitto.pid
log_dest file /Users/liuzhenxing01/mosquitto/mosquitto.log
port 8886
cafile /Users/****/mosquitto/ca.crt
certfile /Users/****/mosquitto/server.crt
keyfile /Users/****/mosquitto/server.key

//如果是单向认证的话次值为false，双向认证的话为true。意思是需要客户端上传客户端证书。
require_certificate true
```

####生成证书

```
https://github.com/MrsSunny/tools/blob/master/TLS/generate-CA.sh执行后会生成CA证书和server证书，然后配置到mosquitto中就可以了。如果使用的是公共的mqtt Broker则不用这个脚本生成证书，直接使用公共的CA证书就可以了。
```

###生成需要的keystore文件

####生成Ca_trust.keystore

```
把上面的文本拷贝到文件然中，命名为mosquitto.org.crt（也可以在test.mosquitto.org上面下载）
如果使用自己部署的服务器则把下面命令的mosquitto.org.crt换成自己的CA证书。

keytool -importcert -trustcacerts -alias test.mosquitto.org -file mosquitto.org.crt -keystore ca_trust.keystore
```

####生成client.keystore

```
openssl genrsa -out client.key
openssl req -out client.csr -key client.key -new
上面两条命令执行完以后会有一个 csr文件，然后把这个文件拷贝到http://test.mosquitto.org/ssl/的form中，然后就会自动下载一个证书（client.crt）。

openssl pkcs12 -export -in client.crt -inkey client.key -name cli -out client.p12 (生成p12文件)

keytool -importkeystore -deststorepass 11111111 -destkeystore client.keystore -srckeystore client.p12 -srcstoretype PKCS12（生成keystore文件）。
```

备注：如果生成了client.crt文件，可以验证一下：openssl verify -CAfile mosquitto.org.crt client.crt ，如果出现client.crt: OK说明client.crt生成成功了。

### 单向认证

> 单向认证只需要生成ca_trust.keystore文件就可以了

```
private KeyStore readCaKeyStore() {
        KeyStore keystore = null;
        try {
            FileInputStream is = new FileInputStream(keystorePath);
            keystore = KeyStore.getInstance("JKS");
            String keypwd = PASSWORD;
            keystore.load(is, keypwd.toCharArray());
        } catch (Exception e) {
            e.printStackTrace();
        }
        return keystore;
}
```

### 获取SSLSocketFactory

```
public static SSLSocketFactory getFactory(KeyStore keystore) {
        try {
            TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
            if (Objects.isNull(keystore)) {
                //使用默认
                tmf.init((KeyStore)null);
            } else {
                tmf.init(keystore);
            }
            SSLContext context = SSLContext.getInstance(TLS_V_1_2);
            context.init(null, tmf.getTrustManagers(), null);
            return context.getSocketFactory();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (KeyStoreException e) {
            e.printStackTrace();
        } catch (KeyManagementException e) {
            e.printStackTrace();
        }
        return null;
}
```



### 双向认证

> 双向认证需要生成client.keystore文件，ca_trust.keystore文件

**加载ca证书代码**

```
private KeyStore readCaKeyStore() {
        KeyStore keystore = null;
        try {
            FileInputStream is = new FileInputStream(keystorePath);
            keystore = KeyStore.getInstance("JKS");
            String keypwd = PASSWORD;
            keystore.load(is, keypwd.toCharArray());
        } catch (Exception e) {
            e.printStackTrace();
        }
        return keystore;
}
```

**加载client证书代码**

```
private KeyStore readClientKeyStore() {
        KeyStore keystore = null;
        try {
            FileInputStream is = new FileInputStream(clientKeystorePath);
            keystore = KeyStore.getInstance("jks");
            String keypwd = PASSWORD;
            keystore.load(is, keypwd.toCharArray());
        } catch (Exception e) {
            e.printStackTrace();
        }
        return keystore;
    }
```

注意：此时导入client.keystore不能像导入CA证书那样导入，如果像CA证书那样导入的话那么在keystore 里面的证书的类型是：trustedCertEntry, 但是在构造KeyManager的时候证书类型必须是PrivateKeyEntry，所以必须把crt文件和私钥转换成P12类型格式，然后再导入到keystore,这样双向验证的就能完成了。

```
public static SSLSocketFactory getFactory(KeyStore caKeystore, KeyStore clientKeystore, String keystorePassword) {
        try {
            Security.addProvider(new BouncyCastleProvider());
            TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
            tmf.init(caKeystore);
            KeyManagerFactory kmf = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
            kmf.init(clientKeystore, keystorePassword.toCharArray());
            SSLContext context = SSLContext.getInstance(TLS_V_1_2);
            context.init(kmf.getKeyManagers(), tmf.getTrustManagers(), null);
            KeyManager[] kms = kmf.getKeyManagers();
            return context.getSocketFactory();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (UnrecoverableKeyException e) {
            e.printStackTrace();
        } catch (KeyStoreException e) {
            e.printStackTrace();
        } catch (KeyManagementException e) {
            e.printStackTrace();
        }
        return null;
    }
```

## 测试

说明：ca.keystore和client.keystore就是上面生成的。

src/test/resource下面的文件是：ca.keystore, client.keystore

**MqttConnectionTest.java**

```
import java.io.FileInputStream;
import java.io.IOException;
import java.net.URL;
import java.security.KeyManagementException;
import java.security.KeyStore;
import java.security.KeyStoreException;
import java.security.NoSuchAlgorithmException;
import java.security.Security;
import java.security.UnrecoverableKeyException;
import java.security.cert.CertificateException;
import java.util.Objects;
import java.util.UUID;

import javax.net.ssl.KeyManager;
import javax.net.ssl.KeyManagerFactory;
import javax.net.ssl.SSLContext;
import javax.net.ssl.SSLSocketFactory;
import javax.net.ssl.TrustManagerFactory;

import org.bouncycastle.jce.provider.BouncyCastleProvider;
import org.eclipse.paho.client.mqttv3.MqttException;
import org.junit.Assert;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;

import mockit.integration.junit4.JMockit;

/**
 * keytool -importcert -trustcacerts -alias <aliasName> -file <ca>.crt -keystore ca_trust.keystore
 * openssl pkcs12 -export -in <client>.crt -inkey <client>.key -name cli -out <client>.p12
 * keytool -importkeystore -deststorepass <your password> -destkeystore <keystore> -srckeystore <client>.p12
 * -srcstoretype PKCS12
 * <p>
 * mosquitto_sub -h test.mosquitto.org -p 8884 -t "topic111111" --cafile mosquitto.org.crt --cert client.crt --key
 * client.key
 * mosquitto_sub -h test.mosquitto.org -p 8883 -t "topic111111" --cafile mosquitto.org.crt
 * Created by liuzhenxing01 on 2018/10/9.
 */

@RunWith(JMockit.class)
public class MqttConnectionTest {

    //无加密信道
    private static String U0;

    //双向加密信道
    private static String U1;

    //单向加密信道
    private static String U2;
    private static String CLIENT_ID;

    //ca证书的keystore地址
    private String caKeystorePath;

    //client证书的keystore地址
    private String clientKeystorePath;

    private static String PASSWORD;

    private static final String TLS_V_1_2 = "TLSv1.2";

    /**
     * 使用公共的mqtt服务器，证书可以在官方网站下载
     */
    @Before
    public void setUp() {
        U0 = "tcp://test.mosquitto.org";
        U1 = "ssl://test.mosquitto.org:8883";
        U2 = "ssl://test.mosquitto.org:8884";
        CLIENT_ID = "SunnyMQTTTest";
        URL resPath = this.getClass().getResource("/");
        caKeystorePath = resPath.getPath() + "ca.keystore";
        clientKeystorePath = resPath.getPath() + "client.keystore";
        PASSWORD = "11111111";
    }

    /**
     * 单向认证测试
     *
     * @throws IOException
     * @throws CertificateException
     * @throws NoSuchAlgorithmException
     * @throws KeyStoreException
     * @throws KeyManagementException
     * @throws MqttException
     */
    //    @Test
    public void mqttSendTestWithTLS() throws IOException, CertificateException, NoSuchAlgorithmException,
            KeyStoreException, KeyManagementException, MqttException, InterruptedException {
        SSLSocketFactory factory = getFactory(readCaKeyStore());
        Assert.assertNotNull(factory);
        pubMessage(factory, U1);
    }

    /**
     * 双向认证测试
     *
     * @throws IOException
     * @throws CertificateException
     * @throws NoSuchAlgorithmException
     * @throws KeyStoreException
     * @throws KeyManagementException
     * @throws MqttException
     * @throws InterruptedException
     * @throws UnrecoverableKeyException
     */
    @Test
    public void mqttSendTestWithClientTLS()
            throws IOException, CertificateException, NoSuchAlgorithmException, KeyStoreException,
            KeyManagementException, MqttException, InterruptedException, UnrecoverableKeyException {
        SSLSocketFactory factory = getFactory(readCaKeyStore(), readClientKeyStore(), PASSWORD);
        Assert.assertNotNull(factory);
        pubMessage(factory, U2);
    }

    public void pubMessage(SSLSocketFactory factory, String serverURI) throws MqttException, InterruptedException {
        MqttConnection connection = new MqttConnection(U2, CLIENT_ID + UUID.randomUUID().toString(), null, null,
                factory);
        connection.openConnection();
        //测试发布需要先判断是否连接上远程MQTT Broker
        int i = 10;
        while (i > 0) {
            if (connection.isConnected()) {
                BceIotMessage message = new BceIotMessage();
                message.setPayload("Sunny test mqtt message".getBytes());
                message.setQos(1);
                message.setTopic("topic111111");
                connection.publishMessage(message);
                connection.publishMessage(message);
                connection.publishMessage(message);
                connection.publishMessage(message);
                connection.publishMessage(message);
                connection.publishMessage(message);
                break;
            }
            Thread.sleep(2000);
            i--;
        }
    }

    /**
     * 获取ca keystore文件
     *
     * @return
     */
    private KeyStore readCaKeyStore() {
        KeyStore keystore = null;
        try {
            FileInputStream is = new FileInputStream(caKeystorePath);
            keystore = KeyStore.getInstance("JKS");
            String keypwd = PASSWORD;
            keystore.load(is, keypwd.toCharArray());
        } catch (Exception e) {
            e.printStackTrace();
        }
        return keystore;
    }

    /**
     * 获取client keystore文件
     *
     * @return
     */
    private KeyStore readClientKeyStore() {
        KeyStore keystore = null;
        try {
            FileInputStream is = new FileInputStream(clientKeystorePath);
            keystore = KeyStore.getInstance("jks");
            String keypwd = PASSWORD;
            keystore.load(is, keypwd.toCharArray());
        } catch (Exception e) {
            e.printStackTrace();
        }
        return keystore;
    }

    public static SSLSocketFactory getFactory(KeyStore keystore) {
        try {
            TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
            if (Objects.isNull(keystore)) {
                //使用默认
                tmf.init((KeyStore)null);
            } else {
                tmf.init(keystore);
            }

            SSLContext context = SSLContext.getInstance(TLS_V_1_2);
            context.init(null, tmf.getTrustManagers(), null);
            return context.getSocketFactory();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (KeyStoreException e) {
            e.printStackTrace();
        } catch (KeyManagementException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 双向验证获取SSLSocketFactory
     * @param caKeystore 存放CA证书的Keystore， @See Junit Test
     * @param clientKeystore 存放客户端的证书，有客户生成由百度DuGo平台的CA证书签名
     * @param keystorePassword clientKeystore 的密码
     * @return
     */
    public static SSLSocketFactory getFactory(KeyStore caKeystore, KeyStore clientKeystore, String keystorePassword) {
        try {
            Security.addProvider(new BouncyCastleProvider());
            TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
            tmf.init(caKeystore);
            KeyManagerFactory kmf = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
            kmf.init(clientKeystore, keystorePassword.toCharArray());
            SSLContext context = SSLContext.getInstance(TLS_V_1_2);
            KeyManager[] kms = kmf.getKeyManagers();
            context.init(kms, tmf.getTrustManagers(), null);
            return context.getSocketFactory();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (UnrecoverableKeyException e) {
            e.printStackTrace();
        } catch (KeyStoreException e) {
            e.printStackTrace();
        } catch (KeyManagementException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

**MqttConnection.java**

```

import java.io.IOException;
import java.security.KeyManagementException;
import java.security.KeyStore;
import java.security.KeyStoreException;
import java.security.NoSuchAlgorithmException;
import java.security.Security;
import java.security.UnrecoverableKeyException;
import java.security.cert.CertificateException;
import java.util.Objects;

import javax.net.SocketFactory;
import javax.net.ssl.KeyManager;
import javax.net.ssl.KeyManagerFactory;
import javax.net.ssl.SSLContext;
import javax.net.ssl.SSLSocketFactory;
import javax.net.ssl.TrustManagerFactory;

import org.bouncycastle.jce.provider.BouncyCastleProvider;
import org.eclipse.paho.client.mqttv3.IMqttActionListener;
import org.eclipse.paho.client.mqttv3.IMqttToken;
import org.eclipse.paho.client.mqttv3.MqttAsyncClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.MqttException;
import org.eclipse.paho.client.mqttv3.MqttMessage;
import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;

/**
 * Created by sunny on 2018/10/9.
 */
public class MqttConnection {

    private MqttAsyncClient mqttAsyncClient;
    private MqttConnectOptions connectionOptions;
    private IMqttActionListener qttActionListener;
    private MqttMessageListener mqttMessageListener;
    private static final String TLS_V_1_2 = "TLSv1.2";
    private BceIotMqttListener bceIotMqttListener;

    public MqttConnection(String serverURI, String clientId, String userName, String
            password, SocketFactory socketFactory) throws MqttException {
        this.mqttAsyncClient = new MqttAsyncClient(serverURI, clientId, new MemoryPersistence());
        this.mqttAsyncClient.setManualAcks(true);
        this.connectionOptions = new MqttConnectOptions();
        this.initOptions(userName, password, socketFactory);
        this.bceIotMqttListener = new BceIotMqttListener(this.mqttAsyncClient);
        this.mqttAsyncClient.setCallback(bceIotMqttListener);
        this.mqttMessageListener = new MqttMessageListener();
    }

    private void initOptions(String userName, String password, SocketFactory socketFactory) {
        this.connectionOptions.setKeepAliveInterval(180);
        this.connectionOptions.setCleanSession(true);
        this.connectionOptions.setSocketFactory(socketFactory);
        if (password != null && !password.isEmpty()) {
            this.connectionOptions.setPassword(password.toCharArray());
        }
    }

    public MqttAsyncClient getMqttAsyncClient() {
        return this.mqttAsyncClient;
    }

    /**
     * 是否连接成功
     * @return false  不能发送和订阅主题消息。
     */
    public boolean isConnected() {
        if (!Objects.isNull(this.mqttAsyncClient)) {
            return this.mqttAsyncClient.isConnected();
        }
        return false;
    }

    public IMqttToken disconnect() throws MqttException {
        if (this.mqttAsyncClient != null) {
            return this.mqttAsyncClient.disconnect();
        }
        return null;
    }

    public void close() throws MqttException {
        if (this.mqttAsyncClient != null) {
            this.mqttAsyncClient.close();
        }
    }

    public void openConnection()  {
        try {
            this.mqttAsyncClient.connect(connectionOptions);
        } catch (MqttException e) {
            e.printStackTrace();
        }
    }

    /**
     * 发布消息到主题
     * @param message
     */
    public void publishMessage(BceIotMessage message) {
        String topic = message.getTopic();
        MqttMessage mqttMessage = new MqttMessage(message.getPayload());
        mqttMessage.setQos(message.getQos());

        try {
            this.mqttAsyncClient.publish(topic, mqttMessage, message, mqttMessageListener);
        } catch (Exception e) {
            //TODO
            e.printStackTrace();
        }
    }

    /**
     * 订阅主题
     * @param message
     */
    public void subscribeTopic(BceIotMessage message) {
        try {
            this.mqttAsyncClient.subscribe(message.getTopic(), message.getQos(), message, mqttMessageListener);
        } catch (MqttException e) {
            //TODO
            e.printStackTrace();
        }
    }

    public void unsubscribeTopic(BceIotMessage message)  {
        try {
            this.mqttAsyncClient.unsubscribe(message.getTopic(), message, mqttMessageListener);
        } catch (MqttException e) {

        }
    }

    /**
     * 单向认证获取SSLSocketFactory， 如果仅仅是单向认证不需要客户端证书，只需要DuGo的CA证书
     * @param keystore
     * @return
     * @throws NoSuchAlgorithmException
     * @throws KeyStoreException
     * @throws KeyManagementException
     * @throws IOException
     * @throws CertificateException
     */
    public static SSLSocketFactory getFactory(KeyStore keystore) {
        try {
            TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
            if (Objects.isNull(keystore)) {
                //使用默认
                tmf.init((KeyStore)null);
            } else {
                tmf.init(keystore);
            }

            SSLContext context = SSLContext.getInstance(TLS_V_1_2);
            context.init(null, tmf.getTrustManagers(), null);
            return context.getSocketFactory();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (KeyStoreException e) {
            e.printStackTrace();
        } catch (KeyManagementException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 双向验证获取SSLSocketFactory
     * @param caKeystore 存放CA证书的Keystore， @See Junit Test
     * @param clientKeystore 存放客户端的证书，有客户生成由百度DuGo平台的CA证书签名
     * @param keystorePassword clientKeystore 的密码
     * @return
     */
    public static SSLSocketFactory getFactory(KeyStore caKeystore, KeyStore clientKeystore, String keystorePassword) {
        try {
            Security.addProvider(new BouncyCastleProvider());
            TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
            tmf.init(caKeystore);
            KeyManagerFactory kmf = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
            kmf.init(clientKeystore, keystorePassword.toCharArray());
            SSLContext context = SSLContext.getInstance(TLS_V_1_2);
            KeyManager[] kms = kmf.getKeyManagers();
            context.init(kms, tmf.getTrustManagers(), null);
            return context.getSocketFactory();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (UnrecoverableKeyException e) {
            e.printStackTrace();
        } catch (KeyStoreException e) {
            e.printStackTrace();
        } catch (KeyManagementException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

**BceIotMessage.java**

```
public class BceIotMessage {

    private String topic;

    private int qos;

    private byte[] payload;

    public String getTopic() {
        return topic;
    }

    public void setTopic(String topic) {
        this.topic = topic;
    }

    public int getQos() {
        return qos;
    }

    public void setQos(int qos) {
        this.qos = qos;
    }

    public byte[] getPayload() {
        return payload;
    }

    public void setPayload(byte[] payload) {
        this.payload = payload;
    }
}
```

**BceIotMqttListener.java**

```
public class BceIotMqttListener implements MqttCallback {

    private  MqttAsyncClient mqttAsyncClient;

    public BceIotMqttListener(MqttAsyncClient mqttAsyncClient) {
        this.mqttAsyncClient = mqttAsyncClient;
    }

    public void connectionLost(Throwable throwable) {

    }

    public void messageArrived(String topic, MqttMessage mqttMessage) throws Exception {

    }

    public void deliveryComplete(IMqttDeliveryToken iMqttDeliveryToken) {

    }
}
```

**MqttMessageListener.java**

```
public class MqttMessageListener implements IMqttActionListener {

    public void onSuccess(IMqttToken iMqttToken) {
        //TODO
    }

    public void onFailure(IMqttToken iMqttToken, Throwable throwable) {
        //TODO
    }
}

```

##测试订阅

```
//无加密订阅
mosquitto_sub -h test.mosquitto.org -p 8884 -t "topic111111"

//单向认证订阅
mosquitto_sub -h test.mosquitto.org -p 8884 -t "topic111111" --cafile mosquitto.org.crt

//双向认证订阅
mosquitto_sub -h test.mosquitto.org -p 8884 -t "topic111111" --cafile mosquitto.org.crt --cert client.crt --key client.key

就可以看到自己用程序发布的主题的消息了。
```

