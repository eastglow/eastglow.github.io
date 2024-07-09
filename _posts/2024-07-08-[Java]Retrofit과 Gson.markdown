---
layout: post
title:  "[Java]Retrofit과 Gson"
date:   2024-07-08
author: EastGlow
categories: Back-end
---

# 1. 들어가며

사내 레거시 시스템에서 MSA 환경에 있는 서비스를 호출 할 때 retrofit을 이용하고 있다. 여느날과 다름없이 레거시 시스템에서 MSA 환경의 API를 호출하기 위해 개발을 하던 중 한가지 이슈에 봉착하게 되었다.

API를 호출하여 body 데이터를 받아왔는데 내가 설정한 받는 쪽 DTO의 필드에 값이 매핑되지 않는 것이었다. 다년간 Json 상하차를 해온 경험을 밑바탕으로 "아, 또 Json 매핑 문제겠구나"를 떠올리며 `@JsonProperty`를 붙여봤는데 역시나 필드에 값이 매핑되지 않았다.

그래서 retrofit의 호출 과정을 쭉 따라가보니 RestAdapter에서 invokeRequest를 호출하는 것을 볼 수 있는데 이 안에서  `Object convert = RestAdapter.this.converter.fromBody(wrapped, type);`  코드를 볼 수 있다.
> retrofit 버전은 1.9.0이다.  

# 2. 원인은?

이 코드를 디버깅 해보니 아래와 같이 2가지 정도의 원인을 추측할 수 있었다.

1.  `@JsonProperty`  는 jackson library를 참조하여 파싱할 때 유효하다. 만약에 gson을 사용하여 파싱한다면  `@SerializedName`  이 필요하다. (참고:  [https://blog.hodory.dev/2019/06/04/json-property-not-working/](https://blog.hodory.dev/2019/06/04/json-property-not-working/))
2.  RestAdapter가 호출하는 `Convert` 객체의 fromBody 구현체를 따라가보니 GsonConvert를 볼 수 있었고 이곳에선 gson을 통해서 파싱되고 있었다.

## RestAdapter
```
Object convert = RestAdapter.this.converter.fromBody(wrapped, type);
```

## Converter
```
public interface Converter {
    Object fromBody(TypedInput var1, Type var2) throws ConversionException;
 
    TypedOutput toBody(Object var1);
}
```

## GsonConverter
```
import com.google.gson.Gson;
import com.google.gson.JsonParseException;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.OutputStream;
import java.io.UnsupportedEncodingException;
import java.lang.reflect.Type;
import retrofit.mime.MimeUtil;
import retrofit.mime.TypedInput;
import retrofit.mime.TypedOutput;
 
... (중략) ...
 
public Object fromBody(TypedInput body, Type type) throws ConversionException {
    String charset = this.charset;
    if (body.mimeType() != null) {
        charset = MimeUtil.parseCharset(body.mimeType(), charset);
    }
 
    InputStreamReader isr = null;
 
    Object var5;
    try {
        isr = new InputStreamReader(body.in(), charset);
        var5 = this.gson.fromJson(isr, type);
    } catch (IOException var15) {
        throw new ConversionException(var15);
    } catch (JsonParseException var16) {
        throw new ConversionException(var16);
    } finally {
        if (isr != null) {
            try {
                isr.close();
            } catch (IOException var14) {
            }
        }
 
    }
 
    return var5;
}
```

위 코드들에서 볼 수 있듯이 fromBody가 구현된 GsonConverter는 Gson library를 참조하고 있었다. 그래서 Gson을 사용한 데이터 매핑 시 사용하는 annotation인 `@SerializedName`를 달아주어 다시 확인을 해보니 그제서야 제대로 필드가 매핑되는 것을 확인할 수 있었다.
