Android软键盘快捷键（仿UC手机浏览器）
==

 最近项目需求要做一个像UC浏览器搜索模块的快捷输入功能，弹出软件盘时在软键盘上方显示一个快捷输入控件，控件可以输入一些快捷字符，可以滑动选中等。下面将对开发进行分析和记录。    
<br/>
<br/>

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
        
        


2.快捷输入
--
 获取光标的开始**Selection.getSelectionStart()**位置和结束位置**Selection.getSelectionEnd()**。利用String的**replace(start,stop,str)**方法将选中部分替换成快捷字符。

 注意：当替换快捷字符后光标的位置总是默认在字符左侧。需求是光标要在替换字符的最右侧。那么就需要移动光标到右侧。
 使用Selection.setSelection(index)，index为光标的位置，只需要将当前光标的位置加上字符长度就可以了。        



