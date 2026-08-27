---
title: "[SW.Modification] Andorid - Dialer - Digits view custom key handling"
date: 2026-08-24 17:50:00
categories:
- SW.Modification
- Devnotes
---

# Why?

This is a featurephone dialer application change. The view for modified dialer consists of:

* **Dialer inputting area**: User inputs a number, to dial
* **Shortcut area**: Show recents, associated number with input and add to contacts or dial menus

![Dialer layout](/images/device-2026-08-24-160234.png)

A method like onKey can be used to handle this, but this is bad practice as this overwrites every key commands in DialpadFragment.

```java
    @Override
    public boolean onKey(View view, int keyCode, KeyEvent event) {
        if (event.getAction() == KeyEvent.ACTION_DOWN) {
            if (mKpdDtmfEntered == false) {
                switch (keyCode) {
                    case KeyEvent.KEYCODE_1:
                        keyPressed(KeyEvent.KEYCODE_1);
                        mKpdDtmfEntered = true;
                        return true;  // Consume the key
                    case KeyEvent.KEYCODE_2:
                        keyPressed(KeyEvent.KEYCODE_2);
                        mKpdDtmfEntered = true;
                        return true;
                    case KeyEvent.KEYCODE_3:
                        keyPressed(KeyEvent.KEYCODE_3);
                        mKpdDtmfEntered = true;
                        return true;
                    case KeyEvent.KEYCODE_4:
                        keyPressed(KeyEvent.KEYCODE_4);
                        mKpdDtmfEntered = true;
                        return true;
                    case KeyEvent.KEYCODE_5:
                        keyPressed(KeyEvent.KEYCODE_5);
                        mKpdDtmfEntered = true;
                        return true;
                    case KeyEvent.KEYCODE_6:
                        keyPressed(KeyEvent.KEYCODE_6);
                        mKpdDtmfEntered = true;
                        return true;
                    case KeyEvent.KEYCODE_7:
                        keyPressed(KeyEvent.KEYCODE_7);
                        mKpdDtmfEntered = true;
                        return true;
                    case KeyEvent.KEYCODE_8:
                        keyPressed(KeyEvent.KEYCODE_8);
                        mKpdDtmfEntered = true;
                        return true;
                    case KeyEvent.KEYCODE_9:
                        keyPressed(KeyEvent.KEYCODE_9);
                        mKpdDtmfEntered = true;
                        return true;
                    case KeyEvent.KEYCODE_0:
                        keyPressed(KeyEvent.KEYCODE_0);
                        mKpdDtmfEntered = true;
                        return true;
                    case KeyEvent.KEYCODE_POUND:
                        keyPressed(KeyEvent.KEYCODE_POUND);
                        mKpdDtmfEntered = true;
                        return true;
                    case KeyEvent.KEYCODE_STAR:
                        keyPressed(KeyEvent.KEYCODE_STAR);
                        mKpdDtmfEntered = true;
                        return true;
                    default:
                        break;
                }
            }
        } else if (event.getAction() == KeyEvent.ACTION_UP) {
            stopTone();
            mKpdDtmfEntered = false;
            
            // Only consume DTMF keys on key up
            switch (keyCode) {
                case KeyEvent.KEYCODE_0:
                case KeyEvent.KEYCODE_1:
                case KeyEvent.KEYCODE_2:
                case KeyEvent.KEYCODE_3:
                case KeyEvent.KEYCODE_4:
                case KeyEvent.KEYCODE_5:
                case KeyEvent.KEYCODE_6:
                case KeyEvent.KEYCODE_7:
                case KeyEvent.KEYCODE_8:
                case KeyEvent.KEYCODE_9:
                case KeyEvent.KEYCODE_POUND:
                case KeyEvent.KEYCODE_STAR:
                    return true;  // Consume DTMF key release
                default:
                    break;
            }
        }
        //move to item 1 of list
        if (keyCode == KeyEvent.KEYCODE_DPAD_DOWN && event.getAction() == KeyEvent.ACTION_DOWN) {
            final DialtactsActivity activity = (DialtactsActivity) getActivity();
            Log.d(TAG, "onKey DPAD_DOWN -> activity="+activity);
            if (activity == null) return false;
            Log.d(TAG, "onKey DPAD_DOWN -> search focus");
            activity.findViewById(R.id.dialtacts_frame).requestFocus();

            AbsListView absListView = (AbsListView)activity.getSearchList();

            if (absListView==null) return false;

            Log.d(TAG, "onKey DPAD_DOWN -> absListView="+absListView);
            absListView.setSelection(1);
            ((DialtactsActivity) getActivity()).getSoftkeyAdapter().setRSKLabel(isDigitsEmpty() ? SoftkeyAdapter.SOFTKEY_LABEL_BACK : SoftkeyAdapter.SOFTKEY_LABEL_CLEAR);
            return true;
        }
        if (keyCode == KeyEvent.KEYCODE_CALL && event.getAction() == KeyEvent.ACTION_UP) {
            dialButtonPressed();
            return true;
        }
        if (keyCode == KeyEvent.KEYCODE_BACK && event.getAction() == KeyEvent.ACTION_UP) {
            if (isDigitsEmpty()) {
                getActivity().finish();
                System.exit(0);
            } else {
                keyPressed(KeyEvent.KEYCODE_DEL);
            }
            return true;
        }
        return false;
    }
```

We tried moving to onKeyDown, but this will not work, no action happens.

We want to keep default of DialtactsActivity whilst having dialer digits area control for keys.

We first see onCreateView:

```java
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedState) {
        final View fragmentView = inflater.inflate(R.layout.dialpad_fragment, container,
                false);
...
        mDigitsContainer = fragmentView.findViewById(R.id.digits_container);
        mDigits = (EditText) fragmentView.findViewById(R.id.digits);
        mDigits.setKeyListener(UnicodeDialerKeyListener.INSTANCE);
        mDigits.setOnClickListener(this);
        mDigits.setOnKeyListener(this);
        mDigits.setOnLongClickListener(this);
        mDigits.addTextChangedListener(mPhoneSearchQueryTextListener);
        mDigits.setFilters(new InputFilter[]{new InputFilter.LengthFilter(MAX_DIGITS_NUMBER_LENGTH)});
...
    }
```

Our main interest is mDigits fragment. It already sets the key listeners to **this** (the view). Altho the key listener is of UnicodeDialerKeyListener.INSTANCE, which we never want to overwrite.

The only solution is to add our own listeners.

The type of **R.id.digits** is com.android.dialer.dialpad.DigitsEditText:

```xml
            <com.android.dialer.dialpad.DigitsEditText
                android:id="@+id/digits"
                android:layout_width="0dip"
                android:layout_weight="1"
                android:layout_height="match_parent"
                android:paddingLeft="10dp"
                android:gravity="center"
                android:scrollHorizontally="true"
                android:singleLine="true"
                android:textAppearance="@style/DialtactsDigitsTextAppearance"
                android:textColor="@color/dialpad_text_color"
                android:textCursorDrawable="@null"
                android:fontFamily="sans-serif-light"
                android:nextFocusRight="@+id/overflow_menu"
                android:background="@android:color/transparent" />
```

DigitsEditText is class of dialer, extends EditText, see alps/packages/apps/Dialer/src/com/android/dialer/dialpad/DigitsEditText.java.

```java
/*
 * Copyright (C) 2011 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package com.android.dialer.dialpad;

import android.content.Context;
import android.graphics.Rect;
import android.text.InputType;
import android.util.AttributeSet;
import android.view.MotionEvent;
import android.view.inputmethod.InputMethodManager;
import android.widget.EditText;

/**
 * EditText which suppresses IME show up.
 */
public class DigitsEditText extends EditText {
    public DigitsEditText(Context context, AttributeSet attrs) {
        super(context, attrs);
        setInputType(getInputType() | InputType.TYPE_TEXT_FLAG_NO_SUGGESTIONS);
        setShowSoftInputOnFocus(false);
    }

    @Override
    protected void onFocusChanged(boolean focused, int direction, Rect previouslyFocusedRect) {
        super.onFocusChanged(focused, direction, previouslyFocusedRect);
        final InputMethodManager imm = ((InputMethodManager) getContext()
                .getSystemService(Context.INPUT_METHOD_SERVICE));
        if (imm != null && imm.isActive(this)) {
            imm.hideSoftInputFromWindow(getApplicationWindowToken(), 0);
        }
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        final boolean ret = super.onTouchEvent(event);
        // Must be done after super.onTouchEvent()
        final InputMethodManager imm = ((InputMethodManager) getContext()
                .getSystemService(Context.INPUT_METHOD_SERVICE));
        if (imm != null && imm.isActive(this)) {
            imm.hideSoftInputFromWindow(getApplicationWindowToken(), 0);
        }
        return ret;
    }
}
```

EditText has no key functions... EditText -> extends TextView.

```
boolean	onKeyDown(int keyCode, KeyEvent event)
Default implementation of KeyEvent.Callback.onKeyDown(): perform press of the view when KeyEvent.KEYCODE_DPAD_CENTER or KeyEvent.KEYCODE_ENTER is released, if the view is enabled and clickable.
```

```
boolean	onKeyUp(int keyCode, KeyEvent event)
Default implementation of KeyEvent.Callback.onKeyUp(): perform clicking of the view when KeyEvent.KEYCODE_DPAD_CENTER, KeyEvent.KEYCODE_ENTER or KeyEvent.KEYCODE_SPACE is released.
```

TextView -> extends View.

```
boolean	dispatchKeyEvent(KeyEvent event)
Dispatch a key event to the next view on the focus path.
```

# Change...

We can cast mDigits to a DigitsEditText type because we have a class where we can add our changes.

```java
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedState) {
        final View fragmentView = inflater.inflate(R.layout.dialpad_fragment, container,
                false);
...
        mDigits.setFilters(new InputFilter[]{new InputFilter.LengthFilter(MAX_DIGITS_NUMBER_LENGTH)});

        DigitsEditText digitsEditText = (DigitsEditText) mDigits;
...
    }
```

As DigitsEditText can have key handling due to extension, we can implement overrides.

```java
    @Override
    public boolean dispatchKeyEvent(KeyEvent event)
    {
        Log.e("saikikusuo->DigitsEditText", "dispatchKeyEvent event="+event);
        return super.dispatchKeyEvent(event);
    }

    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event)
    {
        Log.e("saikikusuo->DigitsEditText", "onKeyDown keyCode="+keyCode+" event="+event+" repeatCount="+event.getRepeatCount());
        return super.onKeyDown(keyCode, event);
    }

    @Override
    public boolean onKeyUp(int keyCode, KeyEvent event)
    {
        Log.e("saikikusuo->DigitsEditText", "onKeyUp keyCode="+keyCode+" event="+event+" repeatCount="+event.getRepeatCount());
        return super.onKeyUp(keyCode, event);
    }
```

## Implement keydown and keyup changes

It's important to declare listener types, a interface type is used. This is somewhat similar to struct.

```java
    public interface KeyShortPressListener
    {
        void onKeyShortClick(int keyCode);
    }

    private KeyShortPressListener mKeyShortPressListener;
    private KeyShortPressListener mKeyShortPressListenerForUp;
```

The mKeyShortPressListener and mKeyShortPressListenerForUp values are null. To set these in a code, we need some setter functions:

```java
    public void setKeyShortPressListener(KeyShortPressListener KeyShortPressListener)
    {
        mKeyShortPressListener = KeyShortPressListener;
    }

    public void setKeyShortPressListenerForUp(KeyShortPressListener KeyShortPressListener)
    {
        mKeyShortPressListenerForUp = KeyShortPressListener;
    }
```

To use those if set, adding the following changes in onKeyDown and onKeyUp:

```java
    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event)
    {
        Log.e("saikikusuo->DigitsEditText", "onKeyDown keyCode="+keyCode+" event="+event+" repeatCount="+event.getRepeatCount());
        //---->
        if (mKeyShortPressListener != null)
        {
            mKeyShortPressListener.onKeyShortClick(keyCode);
        }
        //<----
        return super.onKeyDown(keyCode, event);
    }

    @Override
    public boolean onKeyUp(int keyCode, KeyEvent event)
    {
        Log.e("saikikusuo->DigitsEditText", "onKeyUp keyCode="+keyCode+" event="+event+" repeatCount="+event.getRepeatCount());
        //---->
        if (mKeyShortPressListenerForUp != null)
        {
            mKeyShortPressListenerForUp.onKeyShortClick(keyCode);
        }
        //<----
        return super.onKeyUp(keyCode, event);
    }
```

## Add changed handler

In DialpadFragment, this is code for our requirements:

```java
    DigitsEditText.KeyShortPressListener mMyHandleShortDialpad = new DigitsEditText.KeyShortPressListener()
    {
        @Override
        public void onKeyShortClick(int keyCode)
        {
            if (keyCode >= KeyEvent.KEYCODE_0 && keyCode <= KeyEvent.KEYCODE_9)
            {
                playTone(ToneGenerator.TONE_DTMF_0 + (keyCode - KeyEvent.KEYCODE_0), 250);
            }
            if (keyCode == KeyEvent.KEYCODE_STAR)
            {
                playTone(ToneGenerator.TONE_DTMF_S, 250);
            }
            if (keyCode == KeyEvent.KEYCODE_POUND)
            {
                playTone(ToneGenerator.TONE_DTMF_P, 250);
            }
            switch (keyCode)
            {
                case KeyEvent.KEYCODE_DPAD_DOWN:
                    final DialtactsActivity activity = (DialtactsActivity) getActivity();
                    Log.d(TAG, "mMyHandleShortDialpad->DPAD_DOWN -> activity="+activity);
                    if (activity == null) return;
                    Log.d(TAG, "mMyHandleShortDialpad->DPAD_DOWN -> search focus");
                    activity.findViewById(R.id.dialtacts_frame).requestFocus();

                    AbsListView absListView = (AbsListView)activity.getSearchList();

                    if (absListView==null) return;

                    Log.d(TAG, "mMyHandleShortDialpad->DPAD_DOWN -> absListView="+absListView);
                    absListView.setSelection(1);
                    ((DialtactsActivity) getActivity()).getSoftkeyAdapter().setRSKLabel(isDigitsEmpty() ? SoftkeyAdapter.SOFTKEY_LABEL_BACK : SoftkeyAdapter.SOFTKEY_LABEL_CLEAR);
                    break;
            }
        }
    };

    DigitsEditText.KeyShortPressListener mMyHandleShortDialpadUp = new DigitsEditText.KeyShortPressListener()
    {
        @Override
        public void onKeyShortClick(int keyCode)
        {
            switch (keyCode)
            {
                case KeyEvent.KEYCODE_CALL:
                    dialButtonPressed();
                    break;
                case KeyEvent.KEYCODE_BACK:
                    if (isDigitsEmpty())
                    {
                        ((DialtactsActivity) getActivity()).moveTaskToBack(false);
                    }
                    else
                    {
                        int selectionStart = mDigits.getSelectionStart();
                        int selectionEnd = mDigits.getSelectionEnd();
                        if (selectionStart != -1) {
                            if (selectionStart > selectionEnd) 
                            {
                                int tmp = selectionStart;
                                selectionStart = selectionEnd;
                                selectionEnd = tmp;
                            }
                            if (selectionStart != selectionEnd)
                            {
                                mDigits.getText().delete(selectionStart, selectionEnd);
                            }
                            else
                            {
                                mDigits.getText().delete(selectionStart - 1, selectionStart);
                            }
                        }
                    }
                    break;
            }
        }
    };
```

Comapring original implementation and this change:

* selected text and current text is deleted ->back key and digits !empty
* DTMF done from range for number keys 0-9

## Implement change

```java
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedState) {
        final View fragmentView = inflater.inflate(R.layout.dialpad_fragment, container,
                false);
...
        DigitsEditText digitsEditText = (DigitsEditText) mDigits;
        digitsEditText.setKeyShortPressListener(mMyHandleShortDialpad);
        digitsEditText.setKeyShortPressListenerForUp(mMyHandleShortDialpadUp);
...
    }
```

![Dialer testing 1](/images/20260824-1707.gif)

## Long pressing

Handling of longpress is done by Handler for message and boolean to tell if we are long pressed.

```java
    private Handler handler;
    private boolean lockLongPressKey = false;
```

Do another interface for longpress, then setter:

```java
    public interface KeyLongPressListener
    {
        void onKeyLongClick(int keyCode);
    }

    private KeyLongPressListener mKeyLongPressListener;

    public void setKeyLongPressListener(KeyLongPressListener KeyLongPressListener)
    {
        mKeyLongPressListener = KeyLongPressListener;
    }    
```

On create of the class, handler must handle message. We use message 777 for telling if we are long pressed, then call the longpress listener.

The first argument of message is keycode.

```java
    public DigitsEditText(Context context, AttributeSet attrs) {
        super(context, attrs);
        //<-----
        handler = new Handler() {
            @Override
            public void handleMessage(Message msg) {
                if (msg.what == 777 && DigitsEditText.this.mKeyLongPressListener != null) {
                    DigitsEditText.this.mKeyLongPressListener.onKeyLongClick(msg.arg1);
                }
            }
        };
        //----->
        setInputType(getInputType() | InputType.TYPE_TEXT_FLAG_NO_SUGGESTIONS);
        setShowSoftInputOnFocus(false);
    }
```

Keydown and keyup handling must be changed to handle them.

To tell if long pressed, we get repeatcount of the event.

```
getRepeatCount
Added in API level 1

public final int getRepeatCount ()
Retrieve the repeat count of the event. For ACTION_DOWN events, this is the number of times the key has repeated with the first down starting at 0 and counting up from there. For ACTION_UP events, this is always 0.
```

The lockLongPressKey state must change when repeatcount is > 0, when it is not engaged, we trigger the message and engage setting, after 500ms longkey handle is triggered.

```
sendMessageDelayed
Added in API level 1

public final boolean sendMessageDelayed (Message msg, long delayMillis)
Enqueue a message into the message queue after all pending messages before (current time + delayMillis). You will receive it in handleMessage(Message), in the thread attached to this handler.
```

```java
    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event)
    {
        //<-----
        Log.e("saikikusuo->DigitsEditText", "onKeyDown keyCode="+keyCode+" event="+event+" repeatCount="+event.getRepeatCount()+" lockLongPressKey="+lockLongPressKey);
        if (event.getRepeatCount() > 0 && !lockLongPressKey)
        {
            event.startTracking();
            lockLongPressKey = true;
            Message msg = new Message();
            msg.what = 777;
            msg.arg1 = keyCode;
            handler.sendMessageDelayed(msg, 500L);
            return true;
        }
        //----->
        if (mKeyShortPressListener != null)
        {
            mKeyShortPressListener.onKeyShortClick(keyCode);
        }
        return super.onKeyDown(keyCode, event);
    }

    @Override
    public boolean onKeyUp(int keyCode, KeyEvent event)
    {
        //<-----
        Log.e("saikikusuo->DigitsEditText", "onKeyUp keyCode="+keyCode+" event="+event+" repeatCount="+event.getRepeatCount()+" lockLongPressKey="+lockLongPressKey);
        if (lockLongPressKey)
        {
            lockLongPressKey = false;
            handler.removeMessages(777);
            return true;
        }
        //----->
        if (mKeyShortPressListenerForUp != null)
        {
            mKeyShortPressListenerForUp.onKeyShortClick(keyCode);
        }
        return super.onKeyUp(keyCode, event);
    }
```

For result return value info:

```
Returns
boolean - If you handled the event, return true. If you want to allow the event to be handled by the next receiver, return false.
```

As we are controlling dial digits view (example) from long press only, we need to handle dispatching of the view.

```
Returns
boolean	- True if the event was handled, false otherwise.
```

We filter only 0-9 * # BACK key and longpressing state for assuming the event was handled.

```java
    @Override
    public boolean dispatchKeyEvent(KeyEvent event)
    {
        if (event.getRepeatCount() > 0 && lockLongPressKey && ((event.getKeyCode() >= KeyEvent.KEYCODE_0 && event.getKeyCode() <= KeyEvent.KEYCODE_9)
         || event.getKeyCode() == KeyEvent.KEYCODE_POUND || event.getKeyCode() == KeyEvent.KEYCODE_STAR || event.getKeyCode() == KeyEvent.KEYCODE_BACK))
        {
            return true;
        }
        return super.dispatchKeyEvent(event);
    }
```

## Add changed handler (Long press)

```java
    DigitsEditText.KeyLongPressListener mMyHandleLongDialpad = new DigitsEditText.KeyLongPressListener()
    {
        @Override
        public void onKeyLongClick(int keyCode)
        {
            switch (keyCode)
            {
                case KeyEvent.KEYCODE_BACK:
                    mDigits.setText(null);
                    break;
            }
        }
    };
```

The required change is to clear the digits whenever we have long back press. Add setKeyLongPressListener and set to our digits view.

![Dialer testing 2](/images/20260824-1736.gif)

# Source

https://developer.android.com/reference/android/widget/EditText

https://developer.android.com/reference/android/widget/TextView

https://developer.android.com/reference/android/view/KeyEvent

https://developer.android.com/reference/android/os/Handler