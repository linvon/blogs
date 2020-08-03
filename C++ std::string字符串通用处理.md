---
title: "C++ std::string\u5B57\u7B26\u4E32\u901A\u7528\u5904\u7406"
date: "2019-09-16"
toc: false
tags:
- c++
categories:
- "\u5B66\u4E60\u7B14\u8BB0"
- c++
...
--- ``` c++
/* strutil.h */

#include <string>
#include <vector>
#include <sstream>
#include <iomanip>

// 空格trim处理
std::string trimLeft(const std::string& str);
std::string trimRight(const std::string& str);
std::string trim(const std::string& str);

// 大小写统一转换
std::string toLower(const std::string& str);
std::string toUpper(const std::string& str);

// 首、尾、大小写无关比较
bool startsWith(const std::string& str, const std::string& substr);
bool endsWith(const std::string& str, const std::string& substr);
bool equalsIgnoreCase(const std::string& str1, const std::string& str2);

// 解析字符串
template<class T> T parseString(const std::string& str);
template<class T> T parseHexString(const std::string& str);
template<bool> bool parseString(const std::string& str);

// 格式化为字符串
template<class T> std::string toString(const T& value);
template<class T> std::string toHexString(const T& value, int width = 0);
std::string toString(const bool& value);

// 字符串分割
std::vector<std::string> split(const std::string& str, const std::string& delimiters);

// Tokenizer class
class Tokenizer
{
public:
    static const std::string DEFAULT_DELIMITERS;
    Tokenizer(const std::string& str);
    Tokenizer(const std::string& str, const std::string& delimiters);

    bool nextToken();
    bool nextToken(const std::string& delimiters);
    const std::string getToken() const;

    /**
    * to reset the tokenizer. After reset it, the tokenizer can get
    * the tokens from the first token.
    */
    void reset();

protected:
    size_t m_Offset;
    const std::string m_String;
    std::string m_Token;
    std::string m_Delimiters;
};

// implementation of template functions
template<class T> T parseString(const std::string& str) {
    T value;
    std::istringstream iss(str);
    iss >> value;
    return value;
}

template<class T> T parseHexString(const std::string& str) {
    T value;
    std::istringstream iss(str);
    iss >> hex >> value;
    return value;
}

template<class T> std::string toString(const T& value) {
    std::ostringstream oss;
    oss << value;
    return oss.str();
}

template<class T> std::string toHexString(const T& value, int width) {
    std::ostringstream oss;
    oss << hex;
    if (width > 0) {
        oss << setw(width) << setfill('0');
    }
    oss << value;
    return oss.str();
}
————————————————
版权声明：本文为CSDN博主「边城狂人」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/jamesfancy/article/details/1543338
```

``` c++
/* strutil.cpp */
#include "strutil.h"

#include <algorithm>

using namespace std;

string trimLeft(const string& str) {
    string t = str;
    t.erase(0, t.find_first_not_of(" /t/n/r"));
    return t;
}

string trimRight(const string& str) {
    string t = str;
    t.erase(t.find_last_not_of(" /t/n/r") + 1);
    return t;
}

string trim(const string& str) {
    string t = str;
    t.erase(0, t.find_first_not_of(" /t/n/r"));
    t.erase(t.find_last_not_of(" /t/n/r") + 1);
    return t;
}

string toLower(const string& str) {
    string t = str;
    transform(t.begin(), t.end(), t.begin(), tolower);
    return t;
}

string toUpper(const string& str) {
    string t = str;
    transform(t.begin(), t.end(), t.begin(), toupper);
    return t;
}

bool startsWith(const string& str, const string& substr) {
    if (str.size() < substr.size())
        return false;
    return str.compare(0, substr.size(), substr) == 0;
}

bool endsWith(const string& str, const string& substr) {
    if (str.size() < substr.size())
        return false;
    return str.compare(str.size() - substr.size(), substr.size(), substr) == 0;
}

bool equalsIgnoreCase(const string& str1, const string& str2) {
    return toLower(str1) == toLower(str2);
}

template<bool>
bool parseString(const std::string& str) {
    bool value;
    std::istringstream iss(str);
    iss >> boolalpha >> value;
    return value;
}

string toString(const bool& value) {
    ostringstream oss;
    oss << boolalpha << value;
    return oss.str();
}

vector<string> split(const string& str, const string& delimiters) {
    vector<string> ss;

    Tokenizer tokenizer(str, delimiters);
    while (tokenizer.nextToken()) {
        ss.push_back(tokenizer.getToken());
    }

    return ss;
}

const string Tokenizer::DEFAULT_DELIMITERS("  ");

Tokenizer::Tokenizer(const std::string& str)
    : m_String(str), m_Offset(0), m_Delimiters(DEFAULT_DELIMITERS) {}

Tokenizer::Tokenizer(const std::string& str, const std::string& delimiters)
    : m_String(str), m_Offset(0), m_Delimiters(delimiters) {}

bool Tokenizer::nextToken() {
    return nextToken(m_Delimiters);
}

bool Tokenizer::nextToken(const std::string& delimiters) {
    // find the start charater of the next token.
    size_t i = m_String.find_first_not_of(delimiters, m_Offset);
    if (i == string::npos) {
        m_Offset = m_String.length();
        return false;
    }

    // find the end of the token.
    size_t j = m_String.find_first_of(delimiters, i);
    if (j == string::npos) {
        m_Token = m_String.substr(i);
        m_Offset = m_String.length();
        return true;
    }

    // to intercept the token and save current position
    m_Token = m_String.substr(i, j - i);
    m_Offset = j;
    return true;
}

const string Tokenizer::getToken() const {
    return m_Token;
}

void Tokenizer::reset() {
    m_Offset = 0;
}

————————————————
版权声明：本文为CSDN博主「边城狂人」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/jamesfancy/article/details/1543338
```
