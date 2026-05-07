---
title: 验证工件签名
---

# 签名验证

工件签名允许你验证所拥有的工件是否与项目工作流程创建的工件一致，并且未被未授权方（如中间人攻击）修改。该验证提供了共同基础、保证和知识，确保所有各方引用的是同一工件、同一字节集合，无论是可执行文件、SBOM还是文本文件。

自 Caddy v2.6.0 起，CI/CD 发布工件使用项目 [Sigstore](https://www.sigstore.dev/) 技术进行签名，该技术会签发包含被签发对象详细信息的证书。你可以先检查用于签名所选工件的证书。证书是 base64 编码的，因此首先需要将其 base64 解码以获取 PEM 文件。在此示例中，我们将使用 `caddy_2.6.0_checksums.txt` 工件，并假设在类似 Linux 的环境中进行操作。

首先下载与所选工件相关的三个文件（即 `<工件的实际名称>`——需要验证其附带的签名和证书的工件本身、`<工件名称>.sig`——工件的签名文件、`<工件名称>.pem`——由 Sigstore 的 Fulcio 从根证书派生而来的证书）。然后将下载的 `.pem` 文件 base64 解码为装甲格式：

<pre><code class="cmd bash">base64 -d < caddy_2.6.0_checksums.txt.pem > cert.pem</code></pre>

现在可以使用 `openssl` 命令检查证书。运行 `openssl x509 -in cert.pem -text` 对我们刚刚解码的证书进行操作，会显示如下截断的输出：


<pre><code class="cmd"><span class="bash">openssl x509 -in cert.pem -text</span>
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            22:b0:45:9d:ad:d7:54:98:67:66:b7:de:31:01:ef:4a:02:ab:fb:60
    Signature Algorithm: ecdsa-with-SHA384
        Issuer: O=sigstore.dev, CN=sigstore-intermediate
        Validity
            Not Before: Sep 20 17:17:06 2022 GMT
            Not After : Sep 20 17:27:06 2022 GMT
        Subject:
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:22:ee:f6:b1:85:1c:de:cf:90:1d:91:75:36:c4:
                    82:9d:54:5e:f3:a6:5b:3f:18:89:8a:0b:de:d8:93:
                    7c:02:40:39:00:d4:4e:19:0b:30:93:cc:a4:d0:df:
                    35:f7:b1:08:24:89:cf:3a:38:06:ff:92:75:06:84:
                    b5:9e:25:8c:9a
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature
            X509v3 Extended Key Usage:
                Code Signing
            X509v3 Subject Key Identifier:
                3B:C0:D1:D2:C8:BA:2D:55:95:1F:68:78:DC:C6:2C:D9:B5:17:0E:EA
            X509v3 Authority Key Identifier:
                keyid:DF:D3:E9:CF:56:24:11:96:F9:A8:D8:E9:28:55:A2:C6:2E:18:64:3F

            X509v3 Subject Alternative Name: critical
                URI:https://github.com/caddyserver/caddy/.github/workflows/release.yml@refs/tags/v2.6.0
            1.3.6.1.4.1.57264.1.1:
                https://token.actions.githubusercontent.com
            1.3.6.1.4.1.57264.1.2:
                push
            1.3.6.1.4.1.57264.1.3:
                821a08a6e39ed0e7c43b0271ccf126c194eb6339
            1.3.6.1.4.1.57264.1.4:
                Release
            1.3.6.1.4.1.57264.1.5:
                caddyserver/caddy
            1.3.6.1.4.1.57264.1.6:
                refs/tags/v2.6.0
            1.3.6.1.4.1.11129.2.4.2:
                .z.x.v..`..(R.hE..k'..Eg...=.8.m..".6or....[.DS.....G0E.!..>MD.a..B.p..^..P*...um.....X..F. NYy.....#...TWIZ...y..qa....4P..
    Signature Algorithm: ecdsa-with-SHA384
         30:66:02:31:00:be:b3:3c:15:56:78:64:c6:0f:bc:48:69:a9:
         0a:27:cd:4d:92:39:00:50:42:a8:2a:ad:11:4d:64:f2:61:35:
         ec:08:e9:b5:6a:14:1b:f6:c1:0e:46:ee:a0:54:08:26:e1:02:
         31:00:a7:6d:97:db:4c:c8:dd:47:13:3d:28:7a:a6:f3:64:50:
         2c:5a:9d:9d:10:d0:cf:6f:d0:e9:37:76:fd:cc:8e:9d:c3:6b:
         ba:78:07:40:6a:40:d6:db:f6:97:d5:6a:36:9d
-----BEGIN CERTIFICATE-----
MIIDlDCCAxmgAwIBAgIUIrBFna3XVJhnZrfeMQHvSgKr+2AwCgYIKoZIzj0EAwMw
NzEVMBMGA1UEChMMc2lnc3RvcmUuZGV2MR4wHAYDVQQDExVzaWdzdG9yZS1pbnRl
cm1lZGlhdGUwHhcNMjIwOTIwMTcxNzA2WhcNMjIwOTIwMTcyNzA2WjAAMFkwEwYH
KoZIzj0CAQYIKoZIzj0DAQcDQgAEIu72sYUc3s+QHZF1NsSCnVRe86ZbPxiJigve
2JN8AkA5ANROGQswk8yk0N8197EIJInPOjgG/5J1BoS1niWMmqOCAjgwggI0MA4G
A1UdDwEB/wQEAwIHgDATBgNVHSUEDDAKBggrBgEFBQcDAzAdBgNVHQ4EFgQUO8DR
0si6LVWVH2h43MYs2bUXDuowHwYDVR0jBBgwFoAU39Ppz1YkEZb5qNjpKFWixi4Y
ZD8wYQYDVR0RAQH/BFcwVYZTaHR0cHM6Ly9naXRodWIuY29tL2NhZGR5c2VydmVy
L2NhZGR5Ly5naXRodWIvd29ya2Zsb3dzL3JlbGVhc2UueW1sQHJlZnMvdGFncy92
Mi42LjAwOQYKKwYBBAGDvzABAQQraHR0cHM6Ly90b2tlbi5hY3Rpb25zLmdpdGh1
YnVzZXJjb250ZW50LmNvbTASBgorBgEEAYO/MAECBARwdXNoMDYGCisGAQQBg78w
AQMEKDgyMWEwOGE2ZTM5ZWQwZTdjNDNiMDI3MWNjZjEyNmMxOTRlYjYzMzkwFQYK
KwYBBAGDvzABBAQHUmVsZWFzZTAfBgorBgEEAYO/MAEFBBFjYWRkeXNlcnZlci9j
YWRkeTAeBgorBgEEAYO/MAEGBBByZWZzL3RhZ3MvdjIuNi4wMIGKBgorBgEEAdZ5
AgQCBHwEegB4AHYACGCS8ChS/2hF0dFrJ4ScRWcYrBY9wzjSbea8IgY2b3IAAAGD
W+dEUwAABAMARzBFAiEAnD5NRKZhFLhCHHDIzV6bwVAqlYP6dW0CwKWDo1jzmEYC
IE5ZeeK14oi6I+7z2VRXSVq4/r15GAFxYaCMFrI0UOjjMAoGCCqGSM49BAMDA2kA
MGYCMQC+szwVVnhkxg+8SGmpCifNTZI5AFBCqCqtEU1k8mE17AjptWoUG/bBDkbu
oFQIJuECMQCnbZfbTMjdRxM9KHqm82RQLFqdnRDQz2/Q6Td2/cyOncNrungHQGpA
1tv2l9VqNp0=
-----END CERTIFICATE-----
</code></pre>

<aside class="tip" id="x509-extensions">

注意证书的既定用途为 `Code Signing`（代码签名）。证书还包含在 `X509v3 Subject Alternative Name` 扩展中触发该操作的 GitHub Actions 工作流的 URI，在 `1.3.6.1.4.1.57264.1.4` 中的 GHA 工作流名称，在 `1.3.6.1.4.1.57264.1.3` 中待签名的提交，在 `1.3.6.1.4.1.57264.1.5` 中的仓库名称，以及在 `1.3.6.1.4.1.57264.1.6` 中的触发 ref。这些详细信息共同确定了该证书所针对的全局唯一事件。

</aside>

现在我们有了证书，可以使用 `cosign` CLI 来验证签名。运行以下命令（注意它使用的是未解码的证书）：

<pre><code class="cmd"><span class="bash">COSIGN_EXPERIMENTAL=1 cosign verify-blob --certificate ./caddy_2.6.0_checksums.txt.pem --signature ./caddy_2.6.0_checksums.txt.sig ./caddy_2.6.0_checksums.txt</span>
tlog entry verified with uuid: 04deb84e5a73ba75ea69092c6d700eaeb869c29cae3e0cf98dbfef871361ed09 index: 3618623
Verified OK
</code></pre>

现在切换 CLI 工具，使用 `rekor-cli` 与存储透明度日志的公共 Rekor 服务器交互。运行：

<pre><code class="cmd bash">rekor-cli get --uuid 04deb84e5a73ba75ea69092c6d700eaeb869c29cae3e0cf98dbfef871361ed09 --format json | jq -r '.'
</code></pre>

使用 `jq` 是为了美化输出。你应该会看到类似这样的输出：

```json
{
  "Attestation": "",
  "AttestationType": "",
  "Body": {
    "HashedRekordObj": {
      "data": {
        "hash": {
          "algorithm": "sha256",
          "value": "508f1044ecd9f14c43c6c8986b45b90fc79f25736e2bc85c0911433ce82533f2"
        }
      },
      "signature": {
        "content": "MEUCIHGL2HP5XzcUESTxIk72FS1aNK54LesTfyo+dVhRMeduAiEAnWZDZ5Ur44Y9056vr4to2Fb9FteG53eAFotv3fUZ4h4=",
        "publicKey": {
          "content": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURsRENDQXhtZ0F3SUJBZ0lVSXJCRm5hM1hWSmhuWnJmZU1RSHZTZ0tyKzJBd0NnWUlLb1pJemowRUF3TXcKTnpFVk1CTUdBMVVFQ2hNTWMybG5jM1J2Y21VdVpHVjJNUjR3SEFZRFZRUURFeFZ6YVdkemRHOXlaUzFwYm5SbApjbTFsWkdsaGRHVXdIaGNOTWpJd09USXdNVGN4TnpBMldoY05Nakl3T1RJd01UY3lOekEyV2pBQU1Ga3dFd1lICktvWkl6ajBDQVFZSUtvWkl6ajBEQVFjRFFnQUVJdTcyc1lVYzNzK1FIWkYxTnNTQ25WUmU4NlpiUHhpSmlndmUKMkpOOEFrQTVBTlJPR1Fzd2s4eWswTjgxOTdFSUpJblBPamdHLzVKMUJvUzFuaVdNbXFPQ0FqZ3dnZ0kwTUE0RwpBMVVkRHdFQi93UUVBd0lIZ0RBVEJnTlZIU1VFRERBS0JnZ3JCZ0VGQlFjREF6QWRCZ05WSFE0RUZnUVVPOERSCjBzaTZMVldWSDJoNDNNWXMyYlVYRHVvd0h3WURWUjBqQkJnd0ZvQVUzOVBwejFZa0VaYjVxTmpwS0ZXaXhpNFkKWkQ4d1lRWURWUjBSQVFIL0JGY3dWWVpUYUhSMGNITTZMeTluYVhSb2RXSXVZMjl0TDJOaFpHUjVjMlZ5ZG1WeQpMMk5oWkdSNUx5NW5hWFJvZFdJdmQyOXlhMlpzYjNkekwzSmxiR1ZoYzJVdWVXMXNRSEpsWm5NdmRHRm5jeTkyCk1pNDJMakF3T1FZS0t3WUJCQUdEdnpBQkFRUXJhSFIwY0hNNkx5OTBiMnRsYmk1aFkzUnBiMjV6TG1kcGRHaDEKWW5WelpYSmpiMjUwWlc1MExtTnZiVEFTQmdvckJnRUVBWU8vTUFFQ0JBUndkWE5vTURZR0Npc0dBUVFCZzc4dwpBUU1FS0RneU1XRXdPR0UyWlRNNVpXUXdaVGRqTkROaU1ESTNNV05qWmpFeU5tTXhPVFJsWWpZek16a3dGUVlLCkt3WUJCQUdEdnpBQkJBUUhVbVZzWldGelpUQWZCZ29yQmdFRUFZTy9NQUVGQkJGallXUmtlWE5sY25abGNpOWoKWVdSa2VUQWVCZ29yQmdFRUFZTy9NQUVHQkJCeVpXWnpMM1JoWjNNdmRqSXVOaTR3TUlHS0Jnb3JCZ0VFQWRaNQpBZ1FDQkh3RWVnQjRBSFlBQ0dDUzhDaFMvMmhGMGRGcko0U2NSV2NZckJZOXd6alNiZWE4SWdZMmIzSUFBQUdEClcrZEVVd0FBQkFNQVJ6QkZBaUVBbkQ1TlJLWmhGTGhDSEhESXpWNmJ3VkFxbFlQNmRXMEN3S1dEbzFqem1FWUMKSUU1WmVlSzE0b2k2SSs3ejJWUlhTVnE0L3IxNUdBRnhZYUNNRnJJMFVPampNQW9HQ0NxR1NNNDlCQU1EQTJrQQpNR1lDTVFDK3N6d1ZWbmhreGcrOFNHbXBDaWZOVFpJNUFGQkNxQ3F0RVUxazhtRTE3QWpwdFdvVUcvYkJEa2J1Cm9GUUlKdUVDTVFDbmJaZmJUTWpkUnhNOUtIcW04MlJRTEZxZG5SRFF6Mi9RNlRkMi9jeU9uY05ydW5nSFFHcEEKMXR2Mmw5VnFOcDA9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K"
        }
      }
    }
  },
  "LogIndex": 3618623,
  "IntegratedTime": 1663694226,
  "UUID": "04deb84e5a73ba75ea69092c6d700eaeb869c29cae3e0cf98dbfef871361ed09",
  "LogID": "c0d23d6ad406973f9559f3ba2d1ca01f84147d8ffc5b8445c224f98b9591801d"
}
```

注意，`.Body.HashedRekordObj.signature.content` 的值与我们的 CI 生成的签名内容一致，该签名可在 `caddy_2.6.0_checksums.txt.sig` 文件中找到。此外，所使用并下载的证书也存储在 Rekor 服务器中，并在响应中的 `.Body.HashedRekordObj.signature.publicKey.content` 中可用，且与 `caddy_2.6.0_checksums.txt.pem` 文件中的字符串匹配。我们可以进一步验证 `.Body.HashedRekordObj.data.hash.value` 是否与命令 `sha256sum ./caddy_2.6.0_checksums.txt` 的输出匹配。至此，我们已经匹配了证书、签名以及（包含归档文件校验和但未包含自身校验和的）文件的校验和；此校验和是通过 Sigstore 生态系统外部提供并记录的）。所有这些都公开记录在透明度日志中，供公众验证。

## 验证工件的真实性

如果你获得一个声称是 Caddy 项目产物的工件，但未提供签名文件或证书，该怎么办？你可以使用 `rekor-cli` 查询 Rekor 服务器查找该工件：

<pre><code class="cmd"><span class="bash">rekor-cli search --artifact ./caddy_2.6.0_checksums.txt --format json | jq -r '.UUIDs[0]'</span>
Found matching entries (listed by UUID):
362f8ecba72f432604deb84e5a73ba75ea69092c6d700eaeb869c29cae3e0cf98dbfef871361ed09</code></pre>

注意 UUID 与之前针对同一文件遇到的 UUID 一致。像上一节一样，我们可以查询 Rekor 获取此 UUID 的条目详细信息：

<pre><code class="cmd bash">rekor-cli get --uuid 04deb84e5a73ba75ea69092c6d700eaeb869c29cae3e0cf98dbfef871361ed09 --format json | jq -r '.'</code></pre>

不过，我们可以通过运行以下一行命令来简化查找过程，将两个单独的命令合并为一个：

<pre><code class="cmd"><span class="bash">rekor-cli get --uuid $(rekor-cli search --artifact ./caddy_2.6.0_checksums.txt --format json | jq -r '.UUIDs[0]') --format json | jq -r '.'</span>
{
  "Attestation": "",
  "AttestationType": "",
  "Body": {
    "HashedRekordObj": {
      "data": {
        "hash": {
          "algorithm": "sha256",
          "value": "508f1044ecd9f14c43c6c8986b45b90fc79f25736e2bc85c0911433ce82533f2"
        }
      },
      "signature": {
        "content": "MEUCIHGL2HP5XzcUESTxIk72FS1aNK54LesTfyo+dVhRMeduAiEAnWZDZ5Ur44Y9056vr4to2Fb9FteG53eAFotv3fUZ4h4=",
        "publicKey": {
          "content": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURsRENDQXhtZ0F3SUJBZ0lVSXJCRm5hM1hWSmhuWnJmZU1RSHZTZ0tyKzJBd0NnWUlLb1pJemowRUF3TXcKTnpFVk1CTUdBMVVFQ2hNTWMybG5jM1J2Y21VdVpHVjJNUjR3SEFZRFZRUURFeFZ6YVdkemRHOXlaUzFwYm5SbApjbTFsWkdsaGRHVXdIaGNOTWpJd09USXdNVGN4TnpBMldoY05Nakl3T1RJd01UY3lOekEyV2pBQU1Ga3dFd1lICktvWkl6ajBDQVFZSUtvWkl6ajBEQVFjRFFnQUVJdTcyc1lVYzNzK1FIWkYxTnNTQ25WUmU4NlpiUHhpSmlndmUKMkpOOEFrQTVBTlJPR1Fzd2s4eWswTjgxOTdFSUpJblBPamdHLzVKMUJvUzFuaVdNbXFPQ0FqZ3dnZ0kwTUE0RwpBMVVkRHdFQi93UUVBd0lIZ0RBVEJnTlZIU1VFRERBS0JnZ3JCZ0VGQlFjREF6QWRCZ05WSFE0RUZnUVVPOERSCjBzaTZMVldWSDJoNDNNWXMyYlVYRHVvd0h3WURWUjBqQkJnd0ZvQVUzOVBwejFZa0VaYjVxTmpwS0ZXaXhpNFkKWkQ4d1lRWURWUjBSQVFIL0JGY3dWWVpUYUhSMGNITTZMeTluYVhSb2RXSXVZMjl0TDJOaFpHUjVjMlZ5ZG1WeQpMMk5oWkdSNUx5NW5hWFJvZFdJdmQyOXlhMlpzYjNkekwzSmxiR1ZoYzJVdWVXMXNRSEpsWm5NdmRHRm5jeTkyCk1pNDJMakF3T1FZS0t3WUJCQUdEdnpBQkFRUXJhSFIwY0hNNkx5OTBiMnRsYmk1aFkzUnBiMjV6TG1kcGRHaDEKWW5WelpYSmpiMjUwWlc1MExtTnZiVEFTQmdvckJnRUVBWU8vTUFFQ0JBUndkWE5vTURZR0Npc0dBUVFCZzc4dwpBUU1FS0RneU1XRXdPR0UyWlRNNVpXUXdaVGRqTkROaU1ESTNNV05qWmpFeU5tTXhPVFJsWWpZek16a3dGUVlLCkt3WUJCQUdEdnpBQkJBUUhVbVZzWldGelpUQWZCZ29yQmdFRUFZTy9NQUVGQkJGallXUmtlWE5sY25abGNpOWoKWVdSa2VUQWVCZ29yQmdFRUFZTy9NQUVHQkJCeVpXWnpMM1JoWjNNdmRqSXVOaTR3TUlHS0Jnb3JCZ0VFQWRaNQpBZ1FDQkh3RWVnQjRBSFlBQ0dDUzhDaFMvMmhGMGRGcko0U2NSV2NZckJZOXd6alNiZWE4SWdZMmIzSUFBQUdEClcrZEVVd0FBQkFNQVJ6QkZBaUVBbkQ1TlJLWmhGTGhDSEhESXpWNmJ3VkFxbFlQNmRXMEN3S1dEbzFqem1FWUMKSUU1WmVlSzE0b2k2SSs3ejJWUlhTVnE0L3IxNUdBRnhZYUNNRnJJMFVPampNQW9HQ0NxR1NNNDlCQU1EQTJrQQpNR1lDTVFDK3N6d1ZWbmhreGcrOFNHbXBDaWZOVFpJNUFGQkNxQ3F0RVUxazhtRTE3QWpwdFdvVUcvYkJEa2J1Cm9GUUlKdUVDTVFDbmJaZmJUTWpkUnhNOUtIcW04MlJRTEZxZG5SRFF6Mi9RNlRkMi9jeU9uY05ydW5nSFFHcEEKMXR2Mmw5VnFOcDA9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K"
        }
      }
    }
  },
  "LogIndex": 3618623,
  "IntegratedTime": 1663694226,
  "UUID": "04deb84e5a73ba75ea69092c6d700eaeb869c29cae3e0cf98dbfef871361ed09",
  "LogID": "c0d23d6ad406973f9559f3ba2d1ca01f84147d8ffc5b8445c224f98b9591801d"
}
</code></pre>

现在我们知道该工件已被签名，且其签名已记录在 Rekor 透明度日志服务器上。下一步是验证签名和工件是否来自 Caddy 项目的 CI/CD 工作流程。我们通过从查询 Rekor 获得的 JSON 中提取公钥，base64 解码为 PEM 文件，然后使用 `openssl` 检查证书来完成此操作。运行以下命令从之前收到的 Rekor 响应中提取证书，base64 解码，并将结果存储到文件中。

<pre>
<code class="cmd"><span class="bash">rekor-cli get --uuid $(rekor-cli search --artifact ./caddy_2.6.0_checksums.txt --format json | jq -r '.UUIDs[0]') --format json | jq -r '.Body.HashedRekordObj.signature.publicKey.content' | base64 -d > cert.pem</span></code>
</pre>

现在使用 `openssl` 检查证书，并注意 `X509v3 extensions` 部分。

<pre><code class="cmd"><span class="bash">openssl x509 -in cert.pem -text</span>
Certificate:
...
        Issuer: O=sigstore.dev, CN=sigstore-intermediate
...
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature
            X509v3 Extended Key Usage:
                Code Signing
            X509v3 Subject Key Identifier:
                3B:C0:D1:D2:C8:BA:2D:55:95:1F:68:78:DC:C6:2C:D9:B5:17:0E:EA
            X509v3 Authority Key Identifier:
                keyid:DF:D3:E9:CF:56:24:11:96:F9:A8:D8:E9:28:55:A2:C6:2E:18:64:3F

            X509v3 Subject Alternative Name: critical
                URI:https://github.com/caddyserver/caddy/.github/workflows/release.yml@refs/tags/v2.6.0
            1.3.6.1.4.1.57264.1.1:
                https://token.actions.githubusercontent.com
            1.3.6.1.4.1.57264.1.2:
                push
            1.3.6.1.4.1.57264.1.3:
                821a08a6e39ed0e7c43b0271ccf126c194eb6339
            1.3.6.1.4.1.57264.1.4:
                Release
            1.3.6.1.4.1.57264.1.5:
                caddyserver/caddy
            1.3.6.1.4.1.57264.1.6:
                refs/tags/v2.6.0
            1.3.6.1.4.1.11129.2.4.2:
                .z.x.v..`..(R.hE..k'..Eg...=.8.m..".6or....[.DS.....G0E.!..>MD.a..B.p..^..P*...um.....X..F. NYy.....#...TWIZ...y..qa....4P..
   ...
</code></pre>

[扩展值](#x509-extensions) 指示了工件的真实性。有关每个扩展的定义，请参考 [Sigstore OID 信息](https://github.com/sigstore/fulcio/blob/a25fb09c3f0561ac43e50357fdfc427e3f0aca4a/docs/oid-info.md)。

## 如果签名未验证怎么办？

签名验证失败表示您手中的工件并非由 GitHub 上 Caddy 项目的 CI/CD 工作流程生成。如果您拥有签名、证书和工件，那么应寻找 `cosign` 报告的成功验证。或者，您可以使用 `rekor-cli` 检查 Rekor 服务器中的条目，验证证书扩展是否具有正确且预期的值，并匹配校验和与签名。不匹配或 Rekor 条目不存在意味着工件要么不是由 Caddy 项目的 CI/CD 生成，要么是在 CI/CD 构建流程、GitHub 发布页面以及交付给您之间的某个环节被篡改。