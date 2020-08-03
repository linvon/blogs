---
title: "多种语言实现IP与Int转换"
date: "2018-12-05"
tags: ["shell","ip","lua","c","c++"]
toc: "true"
categories: ["学习笔记", "网络"]
---
> 备注：本文中所有转换均是主机字节序IP地址的转换，需要网络字节序IP地址处理需自行修改代码。
> 相关资料可参考[这里](https://www.cnblogs.com/52php/p/6114080.html)

# **Shell**

- IP转int

```bash
#!/bin/bash
IP_ADDR=$1
IP_LIST=${IP_ADDR//./ };
read -a IP_ARRAY <<<${IP_LIST};      # 把点分十进制地址拆成数组(read的-a选项表示把输入  读入到数组， 下标从0开始)
echo $(( ${IP_ARRAY[0]}<<24 | ${IP_ARRAY[1]}<<16 | ${IP_ARRAY[2]}<<8 | ${IP_ARRAY[3]} ));
```

- Int转IP

```bash
 #!/bin/bash
 N=$1
 H1=$(($N & 0x000000ff))
 H2=$((($N & 0x0000ff00) >> 8))
 L1=$((($N & 0x00ff0000) >> 16))
 L2=$((($N & 0xff000000) >> 24))
 echo $L2.$L1.$H2.$H1
```

- 子网掩码长度转换地址表示

```bash
 #!/bin/bash
 declare -i FULL_MASK_INT=4294967295
 declare -i MASK_LEN=$1
 declare -i LEFT_MOVE="32 - ${MASK_LEN}"
 declare -i N="${FULL_MASK_INT} << ${LEFT_MOVE}"
 declare -i H1="$N & 0x000000ff"
 declare -i H2="($N & 0x0000ff00) >> 8"
 declare -i L1="($N & 0x00ff0000) >> 16"
 declare -i L2="($N & 0xff000000) >> 24"
 echo "$L2.$L1.$H2.$H1"
```

- 子网掩码转换成长度

```bash
 awk '$0>0&&$0<33{for(i=0;i++<4;){if(i<=int($0/8))printf i<4?"255.":"255\n";else if (i==int($0/8)+1 && $0%8){for(n=0;n++<$0%8;)a+=2**(8-n);printf i<4?a".":a"\n"}else printf i<4?0".":0"\n"}}' <<<16
```

- 根据IP地址和子网掩码转换CIDR表示的地址

```bash
 ip1=$(echo $1 | awk -F "." '{print $1}')
 ip2=$(echo $1 | awk -F "." '{print $2}')
 ip3=$(echo $1 | awk -F "." '{print $3}')
 ip4=$(echo $1 | awk -F "." '{print $4}')
 mask1=$(echo $2 | awk -F "." '{print $1}')
 mask2=$(echo $2 | awk -F "." '{print $2}')
 mask3=$(echo $2 | awk -F "." '{print $3}')
 mask4=$(echo $2 | awk -F "." '{print $4}')
 gate1=$(($ip1&$mask1))
 gate2=$(($ip2&$mask2))
 gate3=$(($ip3&$mask3))
 gate4=$(($ip4&$mask4))
 len=$(echo "$2" | awk -F. '{l=0;for(i=1;i<=NF;i++){s=0;for(j=7;j>=0;j--){s+=2**j;if(s==$i){l+=8-j}}}print l}')
 echo "$gate1.$gate2.$gate3.$gate4/$len"
```

- 根据CIDR地址计算起始IP地址

```bash
 off=0;
 data=0;
 for part in $(echo $IP|sed 's/[^0-9]/ /g'); do
         if [ $off -lt 4 ]; then
          data=$(($data * 256 + $part));
     else
          mask=$(((1 << (32 - $part)) - 1)); 
         fi;
         off=$(($off + 1))
 done;
 hostval=$(($data & $mask));
 netval=$(($data - $hostval + 1))
 echo $(($netval >> 24)).$((($netval >> 16) % 256)).$((($netval >> 8) % 256)).$((($netval >> 0) % 256))
```

   

# **C语言**

- IP转int

```c
 unsigned int ip2int(char* ipStr){
     unsigned int ipInt = 0;
     int tokenInt = 0;
     char * token;
     token = strtok(ipStr, ".");
     int i = 3;
     while(token != NULL){
         tokenInt = atoi(token);
         ipInt += tokenInt * pow(256, i);
         token = strtok(NULL, ".");
         i--;
     }
     return ipInt;
 }
```

- Int转IP（注意考虑内存泄漏，用出参形式构成函数）

```c
 static void int2ip(uint32_t ipInt, char * dstStr) {
     int tokenInt = 0;
     uint32_t leftValue = ipInt;
     char *ipStr = (char *)malloc(256);
     memset(ipStr, 0, 256);
     char *ipToken = (char *)malloc(256);
     for(int i = 0; i < 4; i++){
         int temp = pow(256, 3 - i);
         tokenInt = leftValue / temp;
         leftValue %= temp;
         snprintf(ipToken, TOKEN_LEN, "%d", tokenInt);
         if(i != 3){
             strcat(ipToken, ".");
         }
         strncat(ipStr, ipToken, strlen(ipToken));
     }
     strcpy(dstStr, ipStr);
     free(ipStr);
     ipStr = NULL;
     free(ipToken);
     ipToken = NULL;
 }
```




# **C++**

- IP转Int

```c++
 void split(const string& s, vector<string>& sv, const char flag = ' ') {
     sv.clear();
     istringstream iss(s);
     string temp;

     while (getline(iss, temp, flag)) {
         sv.push_back(temp);
     }
     return;
 }

 unsigned int ip2int(string& ip){
     vector<string> ipSecs;
     split(ip, ipSecs, '.');
     vector<string>::iterator it = ipSecs.begin();
     unsigned int ipInt = 0;
     int i = 3;
     stringstream ss;
     for (; it != ipSecs.end(); it++){
         int ipSecInt = 0;
         ss << *it;
         ss >> ipSecInt;
         //must ss.clear()
         ss.clear();
         ipInt += ipSecInt * pow(256,i);
         i--;
     }
     return ipInt;
 }
```

- Int转IP

```c++
 string int2ip(unsigned int ipInt){
     string ip;
     string ipSec;
     stringstream ss;
     unsigned int leftValue = ipInt;
     cout<<leftValue<<endl;
     for(int i = 3; i >= 0; i--){
         int temp = pow(256, i);
         int sectionValue = leftValue / temp;
         leftValue %= temp;
         ss << sectionValue;
         ss >> ipSec;
         ss.clear();
         if(i != 0){
             ipSec.append(".");
         }
         ip.append(ipSec);
         ipSec.clear();
     }
     return ip;
 }
```



# **Lua**

- IP转Int

```lua
 function ip2int(ip)
     local t = {}
     for num in string.gmatch(ip, "%d+") do
         t[#t + 1] = tonumber( num )
     end
     local ret = (((t[1]*256+t[2]))*256+t[3])*256+t[4]

     return ret
 end
```

- Int转IP

```lua
 local bit = require("bit")
 function int2ip( ip )

     local n1 = bit.band(bit.rshift(ip, 24),  0x000000FF)
     local n2 = bit.band(bit.rshift(ip, 16),  0x000000FF)
     local n3 = bit.band(bit.rshift(ip, 8), 0x000000FF)
     local n4 = bit.band(bit.rshift(ip, 0), 0x000000FF)

     return string.format("%d.%d.%d.%d", n1, n2, n3, n4)
 end
```