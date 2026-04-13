```cpp
// utf8编码字符串转unicode编码字符串
CString UTF82WCS(const char* szU8)
{
    //预转换，得到所需空间的大小;
    int wcsLen = ::MultiByteToWideChar(CP_UTF8, NULL, szU8, strlen(szU8), NULL, 0);
    //分配空间要给'\0'留个空间，MultiByteToWideChar不会给'\0'空间
    wchar_t* wszString = new wchar_t[wcsLen + 1];
    
    //转换
    ::MultiByteToWideChar(CP_UTF8, NULL, szU8, strlen(szU8), wszString, wcsLen);
    
    //最后加上'\0'
    wszString[wcsLen] = '\0';
    
    CString unicodeString(wszString);
    
    delete[] wszString;
    wszString = NULL;
    
    return unicodeString;
}
```