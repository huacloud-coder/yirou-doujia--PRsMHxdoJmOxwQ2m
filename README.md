
# 问题描述


对Azure上的虚拟机资源，需要进行安全管理。只有指定的IP地址才能够通过RDP/SSH远程到虚拟机上, 有如下几点考虑：


1） 使用Azure Policy服务，扫描订阅中全部的网络安全组(NSG: Network Security Group) 资源


2） 判断入站规则，判断是否是3389, 22端口


3） 判断源地址是否是被允许的IP


4） 对不满足条件的 NSG规则进行审计。提示不合规( Non\-compliant )


![](https://img2024.cnblogs.com/blog/2127802/202408/2127802-20240829202526614-1937803616.png)


 需要满足以上要求的规则，如何来编写呢？


 


# 问题解答


使用Azure Policy可以完成对Azure资源的集中管制，以满足合规需求。为满足以上需求：


## 第一步：检查NSG资源，查看Inbound ， port， source 值对应的属性名


在NSG的Overview页面，点击右上角的“JSON View”链接，查看资源的JSON内容，其中securityRules的内容为：




```
{
    "name": "RDP",
    "id": "/subscriptions/x-x-x-x/resourceGroups/xxxx/providers/Microsoft.Network/networkSecurityGroups/xxxx-xxx/securityRules/RDP",
    "etag": "W/\"xxxxxxx\"",
    "type": "Microsoft.Network/networkSecurityGroups/securityRules",
    "properties": {
        "provisioningState": "Succeeded",
        "protocol": "TCP",
        "sourcePortRange": "*",
        "destinationPortRange": "3389",
        "sourceAddressPrefix": "167.220.0.0/16",
        "destinationAddressPrefix": "*",
        "access": "Allow",
        "priority": 300,
        "direction": "Inbound",
        "sourcePortRanges": [],
        "destinationPortRanges": [],
        "sourceAddressPrefixes": [],
        "destinationAddressPrefixes": []
    }
}
```


其中：


1. 资源类型为 type \= Microsoft.Network/networkSecurityGroups/securityRules
2. inbound的属性名为 direction
3. 3389端口的属性名为destinationPortRange
4. 而IP地址属性名为sourceAddressPrefix


它们，就是组成Policy Rule的内容


 


## 第二步：编写 Policy Rule


Policy的总体框架是：




```
{
  "mode": "All",
  "policyRule": {
    "if": {
       // 需要进行审计的条件
       //1： 资源的类型是 Microsoft.Network/networkSecurityGroups/securityRules 
       //2： 入站规则 Inbound
       //3： 端口是3389 或 22
       //4： 如果不在允许的IP地址列表里，则需要审计
    },
    "then": {
      "effect": "audit"
    }
  },
  "parameters": {
     //输入参数，本例中是允许的IP地址列表
  }
}
```


### 第一个条件：扫描资源的类型为网络安全组的安全规则



"type": "Microsoft.Network/networkSecurityGroups/securityRules"

**转换为Policy语句：**




```
{
  "field": "type",
  "equals": "Microsoft.Network/networkSecurityGroups/securityRules"
}
```




### 第二个条件：判断规则的方向为入站方向


"direction": "Inbound"


**转换为Policy语句：**






```
     {
            "field": "Microsoft.Network/networkSecurityGroups/securityRules/direction",
            "equals": "Inbound"
     }
```




### 第三个条件：判断端口为3389 或 22


"destinationPortRange": "3389"  或 "destinationPortRange": "22"


**转换为Policy语句：**





```
        {
          "anyOf": [
            {
              "field": "Microsoft.Network/networkSecurityGroups/securityRules/destinationPortRange",
              "equals": "22"
            },
            {
              "field": "Microsoft.Network/networkSecurityGroups/securityRules/destinationPortRange",
              "equals": "3389"
            }
          ]
        }
```



### 第四个条件：判断IP地址，是否不在允许的列表中


"sourceAddressPrefix": "167\.220\.0\.0/16"


**转换为Policy语句：**





```
{
   "field": "Microsoft.Network/networkSecurityGroups/securityRules/sourceAddressPrefix",
    "notIn": "[parameters('allowedIPs')]"
 }
```


 



## 第三步：准备参数(允许的IP地址作为输入参数)


因为被允许的IP地址应该是多个，所以准备为一个Array对象, 参数名称为：allowedIPs。


**结构如下：**




```
    "parameters": {
      "allowedIPs": {
        "type": "Array",
        "metadata": {
          "displayName": "Allowed IPs",
          "description": "The list of allowed IPs for resources."
        },
        "defaultValue": [
          "192.168.1.1"，"x.x.x.x"
        ]
      }
    }
```


## 完整的Policy示例：




```
{
    "mode": "All",
    "policyRule": {
        "if": {
            "allOf": [
                {
                    "field": "type",
                    "equals": "Microsoft.Network/networkSecurityGroups/securityRules"
                },
                {
                    "field": "Microsoft.Network/networkSecurityGroups/securityRules/direction",
                    "equals": "Inbound"
                },
                {
                    "anyOf": [
                        {
                            "field": "Microsoft.Network/networkSecurityGroups/securityRules/destinationPortRange",
                            "equals": "22"
                        },
                        {
                            "field": "Microsoft.Network/networkSecurityGroups/securityRules/destinationPortRange",
                            "equals": "3389"
                        }
                    ]
                },
                {
                    "field": "Microsoft.Network/networkSecurityGroups/securityRules/sourceAddressPrefix",
                    "notIn": "[parameters('allowedIPs')]"
                }
            ]
        },
        "then": {
            "effect": "audit"
        }
    },
    "parameters": {
        "allowedIPs": {
            "type": "Array",
            "metadata": {
                "displayName": "Allowed IPs",
                "description": "The list of allowed IPs for resources."
            },
            "defaultValue": [
                "192.168.1.1","x.x.x.x"
            ]
        }
    }
}
```


## 最终效果：


![](https://img2024.cnblogs.com/blog/2127802/202408/2127802-20240829210150810-2064273075.png)


 


 


# 参考资料


Using arrays in conditions： [https://learn.microsoft.com/en\-us/azure/governance/policy/how\-to/author\-policies\-for\-arrays\#using\-arrays\-in\-conditions](https://github.com)deny\-nsg\-inbound\-allow\-all： [https://github.com/Azure/azure\-policy/blob/master/samples/Network/deny\-nsg\-inbound\-allow\-all/azurepolicy.json](https://github.com):[悠兔机场](https://xinnongbo.com)Azure Policy definitions audit effect： [https://learn.microsoft.com/en\-us/azure/governance/policy/concepts/effect\-audit](https://github.com)


 


