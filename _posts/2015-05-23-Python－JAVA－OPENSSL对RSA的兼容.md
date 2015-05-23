---
layout: post
title: Python－JAVA－OPENSSL对RSA的兼容
date: 2015-05-23 13:37:00
disqus: y
---

加解密钥可以用Python 或者 Openssl 生成，生成格式具有一定的差异，需要进行转换才能使用。

通过openssl生成PrivateKey（openssl文档中说生成的是PrivateKey，而实际上也包换公钥，故其实是一个KeyPair），在python中生成的PrivateKey也即此KeyPair，此PrivateKey的格式是PKCS＃1
	
	openssl genrsa -out mykey.pem 2048

通过KeyPair得到公钥，此Publickey可以在Java中使用，但Python中不能使用，因为格式不相同

	openssl rsa -in mykey.pem -pubout -outform PEM -out public_key.PEM

其内容如下

	-----BEGIN PUBLIC KEY-----
	MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAsluL98s9Z2zE49GgtRNo
	OkD5u63piuDJCtikuoOObrJoL3t71Zd7QeMJ9CTBOxrpgTO71NEEzyDgL+DTh0XT
	KL3v0xjmLoiFHNd5PHqTf9fKq3n6wsT2ZRGmWUSFVqGHqdKo9NgQ68022XRN2JzF
	Ap4ULjGS/VlBIs8Y6xDcYgbsP2EwPaMtTjtqsLkltf4ITQXm6sXBRagzYOyVsW0V
	E7a+Wmj3rSg+Nt2E2WFix1sw4sUsAP4YmUoT2z0CpytPXAp9fknMidtaJz2Y9Llv
	LwRxEdPAv55UUU6uTXo1snHDjL414YQDFh/huMbXeaCW9uvdFHcBCeiFR3Y6QMmi
	DwIDAQAB
	-----END PUBLIC KEY-----


以下是Java读取此公钥文件的代码：

       /**
     * reads a public key from a file
     * @param filename name of the file to read
     * @param algorithm is usually RSA
     * @return the read public key
     * @throws Exception
     */
	public PublicKey getPemPublicKey(String filename, String algorithm) throws Exception {
		File f = new File(filename);
		FileInputStream fis = new FileInputStream(f);
		DataInputStream dis = new DataInputStream(fis);
		byte[] keyBytes = new byte[(int) f.length()];
		dis.readFully(keyBytes);
		dis.close();

		String temp = new String(keyBytes);
		String publicKeyPEM = temp.replace("-----BEGIN PUBLIC KEY-----\n", "");
		publicKeyPEM = publicKeyPEM.replace("-----END PUBLIC KEY-----", "");
		
		BASE64Decoder b64 = new BASE64Decoder();
		byte[] decoded = b64.decodeBuffer(publicKeyPEM);

		X509EncodedKeySpec spec = new X509EncodedKeySpec(decoded);
		KeyFactory kf = KeyFactory.getInstance(algorithm);
		return kf.generatePublic(spec);
	}

在Python中能够使用次公钥，则需要通过pyrsa-priv2pub 进行重新生成一个新的格式的公钥（Python-RSA-compatible）

	pyrsa-priv2pub -i myprivatekey.pem -o mypublickey.pem

则公钥可以在Python中使用，其内容如下，可以看出这个和之前的格式还是不一样的：

	-----BEGIN RSA PUBLIC KEY-----
	MIIBCgKCAQEAsluL98s9Z2zE49GgtRNoOkD5u63piuDJCtikuoOObrJoL3t71Zd7
	QeMJ9CTBOxrpgTO71NEEzyDgL+DTh0XTKL3v0xjmLoiFHNd5PHqTf9fKq3n6wsT2
	ZRGmWUSFVqGHqdKo9NgQ68022XRN2JzFAp4ULjGS/VlBIs8Y6xDcYgbsP2EwPaMt
	TjtqsLkltf4ITQXm6sXBRagzYOyVsW0VE7a+Wmj3rSg+Nt2E2WFix1sw4sUsAP4Y
	mUoT2z0CpytPXAp9fknMidtaJz2Y9LlvLwRxEdPAv55UUU6uTXo1snHDjL414YQD
	Fh/huMbXeaCW9uvdFHcBCeiFR3Y6QMmiDwIDAQAB
	-----END RSA PUBLIC KEY-----


以上对于Openssl 和 Python 而言，其私钥都是一样，都可通用，都是PKCS＃1格式。而在JDK中，需要使用PKCS＃8格式，故需要通过openssl 进行转换
生成PKCS#8格式的私钥

	openssl pkcs8 -topk8 -inform PEM -outform PEM -in mykey.pem -out private_key.pem -nocrypt
	
通过JAVA读取此私钥的代码如下：

	public PrivateKey getPemPrivateKey(String filename, String algorithm)
			throws Exception {
		File f = new File(filename);
		FileInputStream fis = new FileInputStream(f);
		DataInputStream dis = new DataInputStream(fis);
		byte[] keyBytes = new byte[(int) f.length()];
		dis.readFully(keyBytes);
		dis.close();

		String temp = new String(keyBytes);
		String privKeyPEM = temp.replace("-----BEGIN PRIVATE KEY-----\n", "");
		privKeyPEM = privKeyPEM.replace("-----END PRIVATE KEY-----", "");
		
		BASE64Decoder b64 = new BASE64Decoder();
		byte[] decoded = b64.decodeBuffer(privKeyPEM);
		
		PKCS8EncodedKeySpec spec = new PKCS8EncodedKeySpec(decoded);
		KeyFactory kf = KeyFactory.getInstance(algorithm);
		return kf.generatePrivate(spec);
	}

## References
1, [openssl-rsa 文档](https://www.openssl.org/docs/apps/rsa.html)
2, [python-rsa 文档](http://stuvel.eu/files/python-rsa-doc/compatibility.html)
