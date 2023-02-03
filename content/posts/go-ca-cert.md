---
title: "使用 Go 生成自签 CA 证书"
date: 2023-01-24
draft: false
Categories: [go]
---

数字证书是一个经证书授权中心数字签名的包含公开密钥拥有者信息以及公开密钥的文件

## 使用 Go 自签发证书

go 的 x509 标准库下有个 Certificate 结构，这个结构就是证书解析后对应的实体，新证书需要先生成秘钥对，然后使用根证书的私钥进行签名，证书和私钥以及公钥这里使用的是pem编码方式

```go
package main

import (
	"crypto/rand"
	"crypto/rsa"
	"crypto/x509"
	"crypto/x509/pkix"
	"encoding/pem"
	"log"
	"math/big"
	mathRand "math/rand"
	"os"
	"time"
)

const (
	CAFile     = "./test/certs/ca.crt"       // CA证书
	CAKey      = "./test/certs/ca.key"       // CA私钥
	ClientFile = "./test/certs/lisi.pem"     // 客户端证书
	ClientKey  = "./test/certs/lisi_key.pem" // 客户端私钥
)

func main() {
	// 解析根证书
	caFile, err := os.ReadFile(CAFile)
	if err != nil {
		log.Fatal(err)
	}
	caBlock, _ := pem.Decode(caFile)
	caCert, err := x509.ParseCertificate(caBlock.Bytes) // CA证书对象
	if err != nil {
		log.Fatal(err)
	}

	// 解析私钥
	keyFile, err := os.ReadFile(CAKey)
	if err != nil {
		log.Fatal(err)
	}
	keyBlock, _ := pem.Decode(keyFile)
	caPriKey, err := x509.ParsePKCS1PrivateKey(keyBlock.Bytes) // 私钥对象
	if err != nil {
		log.Fatal(err)
	}

	// Go 提供了标准库 crypto/x509 给我们提供了 x509 签证的能力，我们可以先通过 x509.Certificate 构建证书签名请求 CSR 然后再进行签证
	// 构建新的证书模板，里面的字段可以根据自己需求填写
	certTemplate := &x509.Certificate{
		SerialNumber: big.NewInt(mathRand.Int63()), // 证书序列号
		Subject: pkix.Name{
			Country: []string{"CN"},
			//Organization:       []string{"填的话这里可以用作用户组"},
			//OrganizationalUnit: []string{"可填课不填"},
			Province:   []string{"beijing"},
			CommonName: "lisi", // CN
			Locality:   []string{"beijing"},
		},
		NotBefore:             time.Now(),                                                                 // 证书有效期开始时间
		NotAfter:              time.Now().AddDate(1, 0, 0),                                                // 证书有效期
		BasicConstraintsValid: true,                                                                       // 基本的有效性约束
		IsCA:                  false,                                                                      // 是否是根证书
		ExtKeyUsage:           []x509.ExtKeyUsage{x509.ExtKeyUsageClientAuth, x509.ExtKeyUsageServerAuth}, // 证书用途(客户端认证，数据加密)
		KeyUsage:              x509.KeyUsageDigitalSignature | x509.KeyUsageDataEncipherment,
		EmailAddresses:        []string{"UserAccount@virtuallain.com"},
	}

	// 生成公私钥秘钥对
	priKey, err := rsa.GenerateKey(rand.Reader, 2048)
	if err != nil {
		log.Fatal(err)
	}

	// 创建证书对象
	clientCert, err := x509.CreateCertificate(rand.Reader, certTemplate, caCert, &priKey.PublicKey, caPriKey)
	if err != nil {
		log.Fatal(err)
	}

	// 编码证书文件和私钥文件
	clientCertPem := &pem.Block{
		Type:  "CERTIFICATE",
		Bytes: clientCert,
	}

	clientCertFile, err := os.OpenFile(ClientFile, os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0600)
	if err != nil {
		log.Fatal(err)
	}
	err = pem.Encode(clientCertFile, clientCertPem)
	if err != nil {
		log.Fatal(err)
	}

	buf := x509.MarshalPKCS1PrivateKey(priKey)
	keyPem := &pem.Block{
		Type:  "PRIVATE KEY",
		Bytes: buf,
	}
	clientKeyFile, _ := os.OpenFile(ClientKey, os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0600)

	err = pem.Encode(clientKeyFile, keyPem)
	if err != nil {
		log.Fatal(err)
	}
}
```



## 使用证书请求k8s api

关联 roleBinding 后测试一下

```bash
curl --cert ./lisi.pem --key ./lisi_key.pem --cacert /etc/kubernetes/pki/ca.crt -s https://192.168.0.111:6443/api/v1/namespaces/default/pods
```
