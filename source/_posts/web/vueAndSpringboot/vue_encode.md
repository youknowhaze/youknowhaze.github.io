---
title: 验证码组件
date: 2020-09-12
categories:
 - Web
tags:
 - vue
 - java
 - 加密
---


## vue+springboot前后端加密传输

### 一、背景

   项目有一个自己的管理系统，因为仅内网访问，且使用人员只有自己项目组的开发、测试及产品人员，所以怎么简单怎么做的，比较简单。这次因为安全责任从北京迁移至项目开发人员所在地，全公司的所有项目组项目进行安全测试，主要是各个组的管理系统及工具系统。其中就存在一个用户名密码加密传输的问题，从这个问题能映射出其他的数据传输安全问题，故修改之，这里举两个简单的例子（自己项目使用的AES256-CBC加密，这里举例base64和AES128）。


### 二、base64加密

#### 1、前端部分

   首先需要安装base64加密的前端组件，vue根目录下执行如下命令：

````
cnpm install --save js-base64
````

   然后在所需要的项目页面中引入base64组件，如下：

````
let Base64 = require('js-base64').Base64;
````

   在需要进行加解密的地方直接引用方法即可，如下：

````
# 加密
Base64.encode('this_is_a_demo_message');

# 解密
Base64.decode('ZGFua29nYWk=');
````


#### 2、后端部分

   后端部分使用就比较简单了，直接使用即可

````
# 注意引入的base64包
import java.util.Base64;

# 解密方法
String newPwd = new String(Base64.getDecoder().decode(password));

# 加密方法
String newPwd = new String(Base64.getEncoder().encode(password.getBytes()));

````


### 三、AES 128加密

#### 1、前端部分

   需要先安装AES加密所需要的组件
````
npm install crypto-js --save
````

   在util文件夹下创建一个新的js文件，命名为 **aesutil.js** , js代码内容如下：

````
import CryptoJS from 'crypto-js/crypto-js'

// 默认的 KEY 与 iv ，可以和后端商议好，只要统一的给16位字符串即可
const KEY = CryptoJS.enc.Utf8.parse("incredibe#Is100%NB");
const IV = CryptoJS.enc.Utf8.parse('auyejDek134Afw95');

// AES加密
export function Encrypt(word, keyStr, ivStr) {
	let key = KEY
	let iv = IV
	if (keyStr) {
		key = CryptoJS.enc.Utf8.parse(keyStr);
		iv = CryptoJS.enc.Utf8.parse(ivStr);
	}
	let srcs = CryptoJS.enc.Utf8.parse(word);
	var encrypted = CryptoJS.AES.encrypt(srcs, key, {
		iv: iv,
		mode: CryptoJS.mode.CBC,
		padding: CryptoJS.pad.ZeroPadding
	});
	return CryptoJS.enc.Base64.stringify(encrypted.ciphertext);
}

// AES 解密
export function Decrypt(word, keyStr, ivStr) {
	let key  = KEY
	let iv = IV
	if (keyStr) {
		key = CryptoJS.enc.Utf8.parse(keyStr);
		iv = CryptoJS.enc.Utf8.parse(ivStr);
	}
	let base64 = CryptoJS.enc.Base64.parse(word);
	let src = CryptoJS.enc.Base64.stringify(base64);
	var decrypt = CryptoJS.AES.decrypt(src, key, {
		iv: iv,
		mode: CryptoJS.mode.CBC,
		padding: CryptoJS.pad.ZeroPadding
	});
	var decryptedStr = decrypt.toString(CryptoJS.enc.Utf8);
	return decryptedStr.toString();
}
````

   在需要使用的页面引入两个方法，如下：

````
import {Encrypt,Decrypt} from '../utils/aes.js';
````

   使用方法十分简单，直接调用方法即可，因为里面判断了参数是否为true，所以若不需要指定另外的key和vi，则只需要传递待加密/解密的字符即可。

````
# 加密
Encrypt(password)
# 解密
Decrypt(password)
````

#### 2、java部分

   创建用于加解密的工具类，注意key和vi与前端保持一致，在需要加解密的地方直接使用即可

````
import org.apache.commons.codec.binary.Base64;
import javax.crypto.Cipher;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.SecretKeySpec;

public class InteractiveAESUtil {
    //使用AES-128-CBC加密模式，key需要为16位,key和iv可以相同！
    private static final String KEY = "incredibe#Is100%NB";
    private static final String IV = "auyejDek134Afw95";

    /**
     * 加密方法
     * @param data  要加密的数据
     * @param key 加密key
     * @param iv 加密iv
     * @return 加密的结果
     * @throws Exception
     */
    private static String encrypt(String data, String key, String iv){
        try {
            Cipher cipher = Cipher.getInstance("AES/CBC/NoPadding");//"算法/模式/补码方式"NoPadding PkcsPadding
            int blockSize = cipher.getBlockSize();
            byte[] dataBytes = data.getBytes();
            int plaintextLength = dataBytes.length;
            if (plaintextLength % blockSize != 0) {
                plaintextLength = plaintextLength + (blockSize - (plaintextLength % blockSize));
            }
            byte[] plaintext = new byte[plaintextLength];
            System.arraycopy(dataBytes, 0, plaintext, 0, dataBytes.length);
            SecretKeySpec keyspec = new SecretKeySpec(key.getBytes(), "AES");
            IvParameterSpec ivspec = new IvParameterSpec(iv.getBytes());
            cipher.init(Cipher.ENCRYPT_MODE, keyspec, ivspec);
            byte[] encrypted = cipher.doFinal(plaintext);
            return new Base64().encodeToString(encrypted);
        } catch (Exception e) {
            DpasLogger.COMMON.error("前后端交互aes加密异常", e);
            return null;
        }
    }

    /**
     * 解密方法
     * @param data 要解密的数据
     * @param key  解密key
     * @param iv 解密iv
     * @return 解密的结果
     * @throws Exception
     */
    private static String desEncrypt(String data, String key, String iv){
        try {
            byte[] encrypted1 = new Base64().decode(data);
            Cipher cipher = Cipher.getInstance("AES/CBC/NoPadding");
            SecretKeySpec keyspec = new SecretKeySpec(key.getBytes(), "AES");
            IvParameterSpec ivspec = new IvParameterSpec(iv.getBytes());
            cipher.init(Cipher.DECRYPT_MODE, keyspec, ivspec);
            byte[] original = cipher.doFinal(encrypted1);
            String originalString = new String(original);
            return originalString;
        } catch (Exception e) {
            DpasLogger.COMMON.error("前后端交互aes解密异常", e);
            return null;
        }
    }

    /**
     * 使用默认的key和iv加密
     * @param data
     * @return
     * @throws Exception
     */
    public static String encrypt(String data){
        return encrypt(data, KEY, IV);
    }

    /**
     * 使用默认的key和iv解密
     * @param data
     * @return
     * @throws Exception
     */
    public static String desEncrypt(String data){
        return desEncrypt(data, KEY, IV);
    }

}
````










