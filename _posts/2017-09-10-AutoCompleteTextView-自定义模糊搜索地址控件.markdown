---
layout:     post
title:      "AutoCompleteTextView 自定义搜索控件"
subtitle:   "AutoCompleteTextView 自定义搜索控件，通过网络获取数据"
date:       2017-09-10 20:15:00
author:     "wanghao"
header-img: "img/post-bg-js-version.jpg"
tags:
    - AutoCompleteTextView
    - Android
    - 笔记
---
> 针对我们平常的搜索框，一个非常常用的组件，我们一般会考虑使用AutoCompleteTextView 作为自定义搜索控件首选,一般我们提供本地的数据，可以直接使用原生组件，但是，如果获取网络数据我们就需要一些变通的手段

## 背景需求
最近公司需要我们在软件中制作一个带有模糊搜索地址的功能界面组件
如图：

![需求界面](http://img.whdreamblog.cn/18-3-11/1938744.jpg)

我们需要在图片的搜索框中输入相应地址的前几位，同时组件开始进行模糊匹配，获得一组地址后直接进行显示

## 代码分析 

自定义搜索view


```java
public class EhiSearchView<T> extends FrameLayout {
    private EhiAutoCompleteTextView acTvSearchBox;
    private AutoAlignTextView tvCancelBtn;
    private ArrayAdapter<T> adapter;
    private List<T> list;

    /**
    * 部分代码省略
    **/
    public EhiSearchView(@NonNull Context context, @Nullable AttributeSet attrs, @AttrRes int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        View.inflate(context, R.layout.layout_search_view, this);
        initView();
    }
    /**
     * 初始化搜索框以及取消按钮
    **/
    private void initView() {
        acTvSearchBox = (EhiAutoCompleteTextView) findViewById(R.id.ac_tv_search_box);
        tvCancelBtn = (AutoAlignTextView) findViewById(R.id.tv_cancel_btn);
    }

    
    public void setAdapter(@NonNull ArrayAdapter<T> adapter) {
    this.adapter = adapter;
    acTvSearchBox.setAdapter(adapter);
    adapter.notifyDataSetChanged();//此处必须添加通知数据更新
    }

    
    public void setCancelBtnListener(final OnClickCancelBtnListener cancelBtnListener) {
        tvCancelBtn.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View v) {
                cancelBtnListener.onClick(v);
            }
        });
    }

    /**
     * 取消点击事件
     */
    public interface OnClickCancelBtnListener {
        void onClick(View view);
    }

    
    /**
     * 监听搜索文字字数该改变
     *
     * @param textWatcher 监听器
     */
    public void setSearchBoxTextWatcher(TextWatcher textWatcher) {
        acTvSearchBox.addTextChangedListener(textWatcher);
    }


}

```
搜索view的使用代码
```java
public class SearchAddressDialog extends EhiNoFixPlaceDialog {
    private EhiSearchView<PoiInfo> searchView;
    private PoiSearch poiSearch;
    private PoiAdapter poiAdapter;
    private static final String CITY_TAG = "city_tag";
    private static final String CLICK_LISTENER_TAG = "click_listener_tag";
    private OnItemClickListener onItemClickListener;
     /**
       * 部分非关键代码省略
      **/
  
    @Override
    public Dialog onCreateDialog(Bundle savedInstanceState) {
        AlertDialog.Builder builder = new AlertDialog.Builder(getActivity());
        Bundle bundle = getArguments();
        LayoutInflater inflater = getActivity().getLayoutInflater();
        View view = inflater.inflate(R.layout.layout_search_address, null);
        searchView = (EhiSearchView<PoiInfo>) view.findViewById(R.id.search_view);
        searchView.setCancelBtnListener(new EhiSearchView.OnClickCancelBtnListener() {
            @Override
            public void onClick(View view) {
                dismiss();
            }
        });
        initMapSearchConfig();
        initSearchViewConfig(city);
        builder.setView(view);
        return builder.create();
    }

    private void initSearchViewConfig(final String city) {
        searchView.setSearchBoxTextWatcher(new TextWatcher() {
            @Override
            public void beforeTextChanged(CharSequence s, int start, int count, int after) {
                //do nothing
            }

            @Override
            public void onTextChanged(CharSequence s, int start, int before, int count) {
                //do nothing
            }

            @Override
            public void afterTextChanged(Editable s) {
                poiSearch.searchInCity(
                        (new PoiCitySearchOption()).city(city).isReturnAddr(true).keyword(s.toString().trim()).pageNum(0));
            }
        });
        searchView.getAcTvSearchBox().setOnItemClickListener(new AdapterView.OnItemClickListener() {
            @Override
            public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
                if (onItemClickListener != null) {
                    onItemClickListener.onItemClick(parent, view, position, id, searchView.getAdapter().getItem(position));
                }
                dismiss();
            }
        });
    }

    private void initMapSearchConfig() {
        poiSearch = PoiSearch.newInstance();
        poiSearch.setOnGetPoiSearchResultListener(new OnGetPoiSearchResultListener() {
            @Override
            public void onGetPoiResult(PoiResult poiResult) {
                if (getActivity() == null || isDetached()) {
                    return;
                }
                if (poiResult.error != SearchResult.ERRORNO.NO_ERROR) {//包含错误不显示
                    return;
                }
                initAdapter(EhiPoiInfo.convertPoiInfo(poiResult.getAllPoi()));
            }

            @Override
            public void onGetPoiDetailResult(PoiDetailResult poiDetailResult) {
                //do nothing
            }

            @Override
            public void onGetPoiIndoorResult(PoiIndoorResult poiIndoorResult) {
                //do nothing
            }
        });
    }

    /**
     * 初始化适配器
     *
     * @param poiInfoList 搜索信息
     */
    private void initAdapter(List<EhiPoiInfo> poiInfoList) {

        poiAdapter = new PoiAdapter(poiInfoList, getActivity());
        searchView.setAdapter(poiAdapter);
        poiAdapter.notifyDataSetChanged();
    }

   
    public static class EhiPoiInfo extends PoiInfo {
        @Override
        public String toString() {
            return this.name;
        }

        static List<EhiPoiInfo> convertPoiInfo(List<PoiInfo> poiInfoList) {
            String json = GsonUtil.toJson(poiInfoList);
            return GsonUtil.fromJsonList(json, EhiPoiInfo.class);
        }
    }
}
```
### 总结代码思路

自定义思路十分简单粗暴：
1. 将AutoCompleteTextView和取消button定义与布局，封装到一个自定义view中
2. 在使用searchView的类中添加TextWatcher()
```java
searchView.setSearchBoxTextWatcher(new TextWatcher() {
            @Override
            public void beforeTextChanged(CharSequence s, int start, int count, int after) {
                //do nothing
            }

            @Override
            public void onTextChanged(CharSequence s, int start, int before, int count) {
                //do nothing
            }
            /**
              * 检测到输入文字直接使用百度的poiSearch进行模糊匹配
             */ 
            @Override
            public void afterTextChanged(Editable s) {
                poiSearch.searchInCity(
                        (new PoiCitySearchOption()).city(city).isReturnAddr(true).keyword(s.toString().trim()).pageNum(0));
            }
        });
```





