---
layout:     post
title:      "利用GsonConverter统一处理网络model"
subtitle:   " \"自定义GsonConverter的部分内容，统一处理网络model\""
date:       2017-05-29 12:00:00
author:     "wanghao"
header-img: "img/post-bg-retrofit.jpeg"
tags:
    - Android
    - 笔记
    - retrofit
---




## 背景

在我开始了公司的内部司机app的开发后，发现了很多重复的繁琐代码（ps:公司的框架是retrofit+rxjava2+okhttp3）
,在定义接口时，以前是这样的：
```java

public class OldResponseModel<T> extends BaseModel {
  //基础model基类  
  private T Data;
  private boolean IsSuccess;
  private int ErrorCode;
  private String Msg;
}

public interface OldDriverApi {

    //旧版为retrofit定义接口
    @GET("BlockOrder/GetDriverScheduleByWorkdayId")
    Observable<OldResponseModel<List<DriverScheduleMdl>>>
    requestDriverScheduleByWorkdayId(@Query("workdayId") int workDayId);
}    

```
## 优化思路
可以看到在定义接口时，为了使用基础的OldResponseModel，在泛型中进行了大量嵌套，可读性差，且费时费力。
于是，我们重写了部分的GsonResponseBodyConverter进行了对于baseModel的统一处理
```java
public class EhiGsonResponseBodyConverter<T> implements Converter<ResponseBody, T> {
    private final Gson gson;
    private Type type;
    private final TypeAdapter<T> adapter;
    private MyLogger myLogger = MyLogger.getLogger(EhiGsonResponseBodyConverter.class.getSimpleName());

    EhiGsonResponseBodyConverter(Gson gson, TypeAdapter<T> adapter, Type type) {
        this.gson = gson;
        this.type = type;
        this.adapter = adapter;
    }

    /**
     * 使用自定义的convert处理两种业务
     * 1.isSuccess为false 抛出自定义的业务异常类ApiException{@link ApiIOException}
     * 2.当isSuccess为true时，将基础的外层的数据能
     *
     * @param value 相应体 包含数据以及相应信息
     * @return 对应的ResponseModel中model数据
     * @throws IOException 1.json解析异常，借助IO异常抛出
     *                     2.IsSuccess = false 抛出业务自定义IO异常{@link ApiIOException}
     *                     3.系统的流关闭等异常
     */
    @Override
    public T convert(ResponseBody value) throws IOException {
        String originalResponse = value.string();//获取原始json
        String realData;
        try {
            JSONObject jb = new JSONObject(originalResponse);
            boolean isSuccess = jb.optBoolean("IsSuccess");
            if (!isSuccess) {//判断 业务接口是否调用成功
                String message = jb.optString("Message");//发生异常，获取异常message
                if (TextUtils.isEmpty(message) || message.equalsIgnoreCase("null")) {//判断message数据是否异常
                    value.close();
                    throw new ApiIOException("发生未知服务器异常");
                }
                value.close();
                throw new ApiIOException(message);
            }
            if (jb.isNull("Data")) {//使用opt方式，当为null时，会返回“null”
                realData = "";
            } else {
                realData = jb.getString("Data").trim();//判断通过，获取data部分
            }
        } catch (JSONException e) {
            myLogger.e(e);
            value.close();
            throw new IOException(e.getMessage());//为兼容框架的异常处理机制，统一使用IO异常以及IO异常的子类
        }

        if (type == String.class) {//当类型为String直接返回，不用json进行解析，避免字符串中的特殊字符引发json解析报错（此处有坑）// 
            T t = (T) realData;
            return t;
        }

        if (TextUtils.isEmpty(realData)) {
            value.close();
            throw new NullResponseDataIOException();
        }
       

        final String charSet = "UTF-8";
        InputStream inputStream = new ByteArrayInputStream(realData.getBytes(charSet));
        Reader reader = new InputStreamReader(inputStream, charSet);
        JsonReader jsonReader = gson.newJsonReader(reader);
        try {
            T t = adapter.read(jsonReader);
            return t;
        } catch (Exception e) {
            myLogger.e(e);
            return null;
        } finally {
            value.close();
        }
    }
}

public class ApiIOException extends IOException {

  public ApiIOException(String errorMessage) {
    super(errorMessage);
  }
}
```
我们通过JSONObject对于json的字符串进行操作，判断data的isSuccess,为true时，
获取data返回给上层，为false时，读取返回的message抛出api类型的异常，由上层进行处理，例如判断异常类型
如果为api异常，弹窗提示message等。

## 优势
1. 我们api的接口类可以直接进行定义,不需要嵌套
```java
@GET("order/list")
    Observable<List<OrderResponse>> getOrderList(@Query("driverNo") String driverNo);
```
2. 我们可以通过继承代码中ApiIOException进行自定义业务类型异常,比如查询接口为空,这一特殊情况就可以单独定义一个空异常类
```java

if (TextUtils.isEmpty(realData)) {
            value.close();
            throw new NullResponseDataIOException();
        }

public class NullResponseDataIOException extends ApiIOException {
  public NullResponseDataIOException(String errorMessage) {
    super(errorMessage);
  }

  public NullResponseDataIOException() {
    super("服务器数据为空");
  }
}        
```
## 踩坑
1. data类型为String时
```java
 if (type == String.class) {//当类型为String直接返回，不用json进行解析，避免字符串中的特殊字符引发json解析报错（此处有坑）// 
            T t = (T) realData;
            return t;
        }
```
直接进行返回，不要再传入统一处理，因为String中包含一些特殊的字符时，gson处理时会报非法字符异常

2. 如果data为boolean，且message只提供baseModel的message时，需要单独进行处理