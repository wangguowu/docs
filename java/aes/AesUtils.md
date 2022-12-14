# AesUtils
![img.png](img.png)
### 加解密
```
/**
 * 加密为16进制表示
 * @param content
 * @return
 */
public String encryptHex(String aesKey, String content) {
    // 构建Aes
    byte[] key = SecureUtil.generateKey(SymmetricAlgorithm.AES.getValue(), aesKey.getBytes(StandardCharsets.UTF_8)).getEncoded();
    AES aes = SecureUtil.aes(key);
    String encryptHex = aes.encryptHex(content);
    // log.info("encryptHex = 【{}】", encryptHex);
    return encryptHex;
}

/**
 * 解密为字符串
 * @param encryptHex
 * @return
 */
public String decryptStr(String aesKey, String encryptHex) {
    // 构建Aes
    byte[] key = SecureUtil.generateKey(SymmetricAlgorithm.AES.getValue(), aesKey.getBytes(StandardCharsets.UTF_8)).getEncoded();
    AES aes = SecureUtil.aes(key);
    String decryptStr = aes.decryptStr(encryptHex, CharsetUtil.CHARSET_UTF_8);
    log.info("decryptStr = 【{}】", decryptStr);
    return decryptStr;
}

public static void main(String[] args) {
    String encodingAesKey = "g7wzqxwygpnh467f";
    // 解密
    String encryptHex = "00a064f1c0facfce2838aac0220072abb9dd9899f16b1b291db31e50d14673d20e66e3423df49686e9722dd42df997d70d8c865f790b667126d3ef8b0253a5c0a31a7eeb707b282bbff3b0385b269bbaacfc8a763e317358044a68e867bb4cf3946cf08490d0528be20205c7ab1eae2d937268357fbb7a3fb3025fddd7edf1a142d7a8c9411ea31a6f55d0b3a245053a73ed749ff033ac3d0207ec3e26809861d3df1b9aa861ff0c25eb1468505da602cec844e73edecf38103309862c31634b26111ba5e2e71e26ed1c0c647bfc4370501ed141ca1a18f6abecbda1e5cf0fcaec2ae950c3ab611592e9b2c999803f9885f84eed80be0aa21893713082462fd7bb18e66bf183f6a290a5d77f7ff84e1d9faee30542ad463e3f5f925f54741ad651e6b7ce6417b5093420b93bf3942614c8406d8833d89e54fe8d1e3aa17988f220baf19b807acbe1884e7f6d905972eca8b82a2ecddd54a2f98f256da513c573cece3d098631b942a7d2951d37958b4e0c6c4fa2d46ae98e4fd43926681c5ccf5c1befac5038205520055996dddb0585626f53b889ca08b1512194a3332c4fab";
    byte[] key = SecureUtil.generateKey(SymmetricAlgorithm.AES.getValue(), encodingAesKey.getBytes(StandardCharsets.UTF_8)).getEncoded();
    AES aes = SecureUtil.aes(key);
    String decryptStr = aes.decryptStr(encryptHex, CharsetUtil.CHARSET_UTF_8);
    log.info("decryptStr = 【{}】", decryptStr);

    // encode
    String enStr = aes.encryptHex(decryptStr);
    System.out.println(enStr);
    System.out.println(enStr.equals(encryptHex));

}
```
微信加解密参考：https://developers.weixin.qq.com/doc/offiaccount/Message_Management/Message_encryption_and_decryption_instructions.html

### apply
```
package com.tiku.web.controller.api;

import cn.hutool.core.util.CharsetUtil;
import cn.hutool.crypto.SecureUtil;
import cn.hutool.crypto.symmetric.AES;
import cn.hutool.crypto.symmetric.SymmetricAlgorithm;
import com.alibaba.fastjson.JSON;
import com.tiku.common.core.controller.BaseController;
import com.tiku.common.core.domain.AjaxResult;
import com.tiku.common.core.redis.RedisCache;
import com.tiku.common.utils.StringUtils;
import com.tiku.open.domain.OpenApp;
import com.tiku.open.service.IOpenAppService;
import com.tiku.system.domain.SysPushLog;
import com.tiku.system.service.ISysPushLogService;
import com.tiku.web.request.api.PullReq;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.core.MessageBuilder;
import org.springframework.amqp.core.MessageProperties;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.nio.charset.StandardCharsets;

import static com.tiku.common.constant.RedisConstants.TOKEN_APPID;

/**
 * 数据拉取
 * @author wgw
 */
@Slf4j
@RestController
@RequestMapping("/api/cert")
public class ApiCertController extends BaseController {

    @Autowired
    private RabbitTemplate rabbitTemplate;
    @Autowired
    private RedisCache redisCache;
    @Autowired
    private IOpenAppService openAppService;
    @Autowired
    private ISysPushLogService sysPushLogService;

    @PostMapping("/pull/{token}")
    public AjaxResult pull(@PathVariable("token") String token, @RequestBody String encryptData) {
        encryptData = StringUtils.substring(encryptData, 0, encryptData.length()-1);
        log.info("token = 【{}】, 收到密文【{}】", token, encryptData);
        String appId = redisCache.getCacheObject(String.format(TOKEN_APPID, token));
        if (StringUtils.isBlank(appId)) {
            return AjaxResult.error(1001, "无效token");
        }

        OpenApp openApp = openAppService.selectOpenAppByAppId(appId);
        String encodingAesKey = openApp.getEncodingAesKey();
        String data = this.decryptStr(encodingAesKey, encryptData);
        log.info("解析密文，得到明文【{}】", data);

        // 取数据 && 加密
        PullReq pullReq = JSON.parseObject(data, PullReq.class);
        SysPushLog pushLog = sysPushLogService.selectSysPushLogByMsgId(pullReq.getMsgId());
        String enStr = this.encryptHex(encodingAesKey, pushLog.getLongData());
        log.info("响应密文【{}】", enStr);
        return AjaxResult.success(enStr);
    }

    @GetMapping("/send/{msgId}")
    public AjaxResult send(@PathVariable("msgId") String msgId) {
        SysPushLog pushLog = sysPushLogService.selectSysPushLogByMsgId(msgId);
        // send msg
        Message sendMsg = MessageBuilder.withBody(pushLog.getShortData().getBytes(StandardCharsets.UTF_8))
                .setMessageId(msgId)
                .setContentType(MessageProperties.CONTENT_TYPE_JSON)
                .setContentEncoding(StandardCharsets.UTF_8.name())
                .build();
        rabbitTemplate.convertAndSend(pushLog.getReceiver(), sendMsg);
        return AjaxResult.success("send");
    }
}


```
