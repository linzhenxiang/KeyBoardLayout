Android软键盘快捷键（仿UC手机浏览器）
==

 最近项目需求要做一个像UC浏览器搜索模块的快捷输入功能，弹出软件盘时在软键盘上方显示一个快捷输入控件，控件可以输入一些快捷字符，可以滑动选中等。下面将对开发进行分析和记录。    
<br/>
<br/>

<img height="800px" width="480px" src="https://gitlab.com/linzhenxiang/KeyBoardLayout/tree/master" />

1.功能分析
--
> * 快捷输入 （点击快捷键，内容将会输入到输入框）
> * 滑块选中 （滑动滑块可以选中文本）
> * 弹出选中菜单（菜单拥有剪切、复制、分享等）
> * 剪切板 （点击快捷键显示剪切板界面)

<br/>


2.功能实现
--
以下介绍各功能的具体实现思路，开发中遇到的问题和解决方法。

1.滑块选中功能
---
>  **A） 按下事件**
>> -  当选中菜单显示时，需要关闭选中菜单 

<br/>

>  **B） 长按事件**
>> -  输入框获取焦点，显示光标（滑块滑动时输入框光标会隐藏，此时需要重新显示光标并保证滑动时光标不在消失

>> -  获取输入框当前光标所在位置，用于后面滑动时选中

>> -  触发滑块拉伸动画

>> -  触发震动事件

<br/>

>  **C） 滑动选中**
>> - 左右滑动选中文本

>> - 文本过长，滑块滑到顶端时文本自动滚动且选中

>> - 反向或结束滑动时，自动滚动立刻结束

<br/>

> **D） 滑动结束**

>> - 滑动结束时若有文本被选中，则弹出选中快捷菜单 

>> - 结束拉伸动画

<br/>

> **E）代码块**

> **1.给滑块设置OnTouchListener监听**

   这里要着重说一下为什么不用**OnLongClickListener**监听长按事件。由于滑**AbsSeekBar**重写了**onTouchEvent()**方法去除了点击和长按事件。只能通过设置**onTouchListener**事件监听触摸事件，而**onTouchListener**是在**dispatchTouchEvent(MotionEvent event)**方法内触发的，事件触发在**onTouchEvent**之前。

        public boolean dispatchTouchEvent(MotionEvent event) {  
        if (!onFilterTouchEventForSecurity(event)) {  
            return false;  
        }  
  
        if (mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED &&  
                mOnTouchListener.onTouch(this, event)) {  
            return true;  
        }  
        return onTouchEvent(event);  
    }  

<br/>
>  **2.onTouch方法回调**

由于滑块的点击和长按事件无法触发，所以只通过手势的**onDown** 和 **onLongPress** 监听按下和长按事件     

       @Override
    public boolean onTouch(View v, MotionEvent event) {
        //手势监听
        mSeekBarGestureDetector.onTouchEvent(event);
        //长按事件结束
        if (event.getAction() == MotionEvent.ACTION_UP && isLongPress) {
            //防抖动，避免多次点击而频繁执行动画
			if (Shake.shake(v)) {
                return false;
            }
			//标记滑块触摸事件结束
            isSeekTouchUp = true;
			//结束滑块的拉伸动画（D2)
            stopSeekAnim(getMeasuredWidth() - (int) (getResources().getDisplayMetrics().density * 32));

        }
        return false;
    }

<br/>
> **3.手势事件回调**

		//滑块按下事件
      @Override
        public boolean onDown(MotionEvent e) {
			//输入框EditText 对象不为
            requireNonNull(mEditText, "editText can't be null");
			//关闭输入框选中选中菜单（当菜单显示时，再次滑动滑块需要先关闭选中菜单，否则菜单会频繁弹出，导致界面重新layot影响滑块动画执行）---(A1)
            EditorCompat.stopTextActionMode(mEditText);
            return super.onDown(e);
        }
		
		//滑块长按事件
        @Override
        public void onLongPress(MotionEvent e) {
            super.onLongPress(e);
            requireNonNull(mEditText, "editText can't be null");
            //触发震动 震动持续100ms ---(B4)
			VibratorCompat.Vibrate(getContext(), 100);
            //获取光标的初始位置，用于文本选中 ---(B2)
			mCursorPosition = mEditText.getSelectionStart();
            //初始化光标（当滑动滑块时，光标会消失，滑动过程中没有选中状态变化，所以需要先初始化一下光标位置，让光标显示且文本处于可选状态）---(B1)
			EditorCompat.positionAtCursorOffset(mEditText);
            //触发滑块的横向拉伸动画 ---(B3)
			startSeekAnim(mSeekBarWidth);

        }


> **4.SeekBar 滑动事件**

在SeekBar的滑动回调中处理滑动选中和文本滚动选中。首先获取滑块的初始位置，SeekBar滑动时获取滑动的差值并更新位置，根据差值移动光标选中文本，当滑块到达顶端而文本过长，此时需要通过Handler 定时发送移动光标选中文本的消息，直到文本全部选中。当反向移动或停止触摸SeekBar时需要立刻取消Handler消息。

	//SeekBar 滑动
    @Override
    public void onProgressChanged(SeekBar seekBar, int progress, boolean fromUser) {
         1. 判断SeekBar是长按状态
		
		 2. 获取滑动差值（int dx = progress -mLastPosition）
		
		 3. 判断差值是否大于0以区分是向左还是向右滑动
		
		 4. 差值<=0 向左滑动，此时取消向右自动滚动选中文本的Handler消息。
			使用Selection的setSelection(text,start,stop)方法选中文本，text为文本内容，
			start是光标开始的位置，stop是光标结束的位置
			当滑块到达左端而光标未到达最左端时，使用Handler延迟150ms发送向左自动滚动选中文本的消息
            
 			差值>0 向右滑动，此时取消向左自动滚动选中文本的Handler消息，
			使用Selection的setSelction(text,start.stop)选中文本
			当滑块到右端而光标未到达最右端时，使用Handler延迟150ms发送向右自动滚动选中文本的消息
		
		5.更新SeekBar的位置
    }

	//SeekBar触摸事件触发
    @Override
    public void onStartTrackingTouch(SeekBar seekBar) {

        //记录当前SeekBar位置 mLastPosition

    }

	//SeekBar 触摸事件结束
    @Override
    public void onStopTrackingTouch(SeekBar seekBar) {
       
		 1. 取消所有Handler消息
		 2. 若SeekBar是点击事件而非长按事件，释放时需要重置SeekBar 初始位置
      
    }


> **5.Handler 消息**

    @Override
    public void handleMessage(Message msg) {
            super.handleMessage(msg);
            switch (msg.what) {
 				//向左滑动
                case 1:
                    //1.移动选中文本
                    //2.延时发送继续向左移动选中文本的消息
				
                    break;
				//向右滑动
                case 2:
					//1.移动选中文本
                    //2.发送继续向右移动选中文本的消息
				
                    break;
            }
        }
        
        


<br/>
<br/>
2.快捷输入
--
 获取光标的开始**Selection.getSelectionStart()**位置和结束位置**Selection.getSelectionEnd()**。利用String的**replace(start,stop,str)**方法将选中部分替换成快捷字符。

 注意：当替换快捷字符后光标的位置总是默认在字符左侧。需求是光标要在替换字符的最右侧。那么就需要移动光标到右侧。
 使用Selection.setSelection(index)，index为光标的位置，只需要将当前光标的位置加上字符长度就可以了。        


<br/>
<br/>
3.弹出和关闭选中快捷菜单
--
当释放**SeekBar**时,需要弹出选中快捷菜单，当然若没有内容被选中菜单是不会弹出的，而在此触摸滑块时需要关闭菜单避免影响滑块的拉伸动画。

**注意：**菜单的弹出玉打开需要通过Java的反射机制打开。这是因为选中快捷菜单是由**TextView**中的**mEditor**对象控制的，而TextView并没有提供外部获取**mEditor**对象的public方法。这就需要我们获取**mEditor**对象，进而获取**Editor**类的私有类对象及该私有类的方法。通过反射机制执行该方法来达到我们想要的效果。
 

<br/>
<br/>

4.剪切板
--
待续-------------------
 



<br/>
<br/>
总结：开发中遇到的一些问题
==

<br/>
>  **SeekBar 样式更改**


- 圆形的滑块周围总是有白色的边框，开上去是在滑块外套了个矩形。

通过设置SeekBar的android:splitTrack="false"解决了问题。这个属性的作用是决定是否分割进度条，我们通常见到的进度条左右颜色不一样就是设定这个属性来分割的，该属性的默认值为TRUE。
我们看一下 这个属性的描素：

     /**
     * Specifies whether the track should be split by the thumb. When true,
     * the thumb's optical bounds will be clipped out of the track drawable,
     * then the thumb will be drawn into the resulting gap.
     *
     * @param splitTrack Whether the track should be split by the thumb
     */
      public void setSplitTrack(boolean splitTrack) {
        mSplitTrack = splitTrack;
        invalidate();
    }


大意：用于指定进度条是否需要分割，若选择分割，那么拇指按钮的可视边界将取决于分割后的进度图片（由于分割是矩形分割，所以会有白色的背景），否则拇指图标将被绘制在进度的间隙内。

<br/>

> **SeekBar 点击和长按事件无法触发**

SeeKBar 继承于AbsSeekBar 而AbsSeekBar 对View的onTouchEvent方法进行了重写，去除了点击和长按事件的处理，因此发触发事件回调。
然而SeekBar的onTouchListener事件是可以正常回调的，因为该事件是在dispatchTouchEvent事件分发方法中进行判断的，方法的调用要在onTouchEvent之前。那么通过GestureDetector 的onTouchEvent方法接收onTouchListener的回到事件就可以实现点击和长按事件的监听。

<br/>
<br/>

> **SeekBar 的动画会卡住导致控件无法完全拉伸或收缩**

1.要标记按下，长按，释放时的状态和防抖动，避免多次触发动画

2.避免界面进行重新layout，在动画开始期间，不能有新的控件显示或隐藏（eg:选中菜单的显示与关闭必须与动画错开)

<br/>
<br/>

> **SeekBar滑动时光标消失，无法看到选中状态**

输入框的焦点还在，光标隐藏了，单纯的再次显示光标并不能解决问题，后来通过反射的方法初始化了光标和编辑状态就可以了。

<br/>
<br/>

> **选中文本时，过多的文本不自动滚动**

文本选中是通过Selection.setSelection（str,start,stop)方法实现的，实现过程中发现光标移动到顶端时就不在移动了。后来发现文本的滚动是选中文本的结束位置决定的，所以当start到达顶端时就不移动了，那么固定start的位置，移动stop的位置就解决的这个问题。在移动过程中要注意方向和移动的距离避免数组越界。

<br/>
> **屏幕休眠再唤醒时发现滑动动画为执行结束**

添加屏幕休眠和唤醒监听，在屏幕休眠时启动滑块的缩放动画。
