# Leetcode

正则表达式的匹配：参考<https://blog.csdn.net/qq_41231926/article/details/82010888>

递归做法。

```c++
class Solution {
public:
    bool isMatch(string s, string p){
        //1 s="xxx" p=""
        if(s.length() != 0 && p.length() == 0)
            return false;
        
        //2 s="" p="a*b*"
        if(s.length() == 0)
        {
            for(int i = 0; i < p.length(); i++)
            {
                if(i % 2 == 1 && p[i] != '*')
                    return false;
            }
            return p.length() % 2 == 0;
        }
        
        //3 s!="" && p!=""
        int si = s.length() - 1;
        int pj = p.length() - 1;
        
        if(p[pj] != '*')//3.1末尾不是*
        {
            if(p[pj] == s[si] || p[pj] == '.')//3.1.1 末尾匹配，则如果前面也匹配则整体匹配
            {
                return isMatch(s.substr(0,si), p.substr(0, pj));
            }
            return false;//3.1.2 末尾不匹配，整体不匹配
        }
        else{//3.2 末尾是*
            if(p[pj - 1] != s[si] && p[pj-1] != '.')
            {//3.2.1 s[len-1]与p[len-2]不匹配，则看看*匹配0位时，如果前面也匹配则整体匹配
                return isMatch(s, p.substr(0, pj - 1));
            }
            else
            {//3.2.2 *匹配1位，则前面剩下s[:len - 1] p[:len - 2]
             //3.2.3 *匹配0位，则前面剩下s[:len] p[:len - 2]
             //3.2.4 *匹配多于1位，则前面剩下s[:len - 1] p[:len]
                return isMatch(s.substr(0, si), p) ||isMatch(s, p.substr(0, pj - 1)) || isMatch(s.substr(0, si), p.substr(0, pj - 1));
            }
        }
        return false;
    }
};
```

