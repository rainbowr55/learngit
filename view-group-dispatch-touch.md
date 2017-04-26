####ViewGroup dispatchTouchEvent 源码学习

常说事件传递中的流程是:dispatchTouchEvent->onInterceptTouchEvent->onTouchEvent
今天来学习ViewGroup是如何通过dispatchTouchEvent分发事件的，在dispatchTouchEvent()决定了Touch事件是由自己的onTouchEvent()处理
今天来学习ViewGroup是如何通过dispatchTouchEvent分发事件的，在dispatchTouchEvent()决定了Touch事件是由自己的onTouchEvent()处理  dispatchTouchEvent()会调用dispatchTransformedTouchEvent()方法且在该方法中递归调用  dispatchTouchEvent();从而会在dispatchTouchEvent()里最终调用到onTouchEvent() 还是分发给子View处理让子View调用其自身的dispatchTouchEvent()处理.
    * 重点关注:
    * 1 子View对于ACTION_DOWN的处理十分重要!!!!!
    *   ACTION_DOWN是一系列Touch事件的开端,如果子View对于该ACTION_DOWN事件在onTouchEvent()中返回了false即未消费.
    *   那么ViewGroup就不会把后续的ACTION_MOVE和ACTION_UP派发给该子View.在这种情况下ViewGroup就和普通的View一样了,
    *   调用该ViewGroup自己的dispatchTouchEvent()从而调用自己的onTouchEvent();即不会将事件分发给子View.
    *   详细代码请参见如下代码分析.
    *   
    * 2 为什么子view对于Touch事件处理返回true那么其上层的ViewGroup就无法处理Touch事件了?????
    *   这个想必大家都知道了,因为该Touch事件被子View消费了其上层的ViewGroup就无法处理该Touch事件了.
    *   那么在源码中的依据是什么呢??请看下面的源码分析
    *   
```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
    }

    // If the event targets the accessibility focused view and this is it, start
    // normal event dispatch. Maybe a descendant is what will handle the click.
    if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
        ev.setTargetAccessibilityFocus(false);
    }

    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        // Handle an initial down. 对ACTION_DOWN进行处理
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Throw away all previous state when starting a new touch gesture.
            // The framework may have dropped the up or cancel event for the previous gesture
            // due to an app switch, ANR, or some other state change.
            //清除以往的Touch状态(state)开始新的手势(gesture)
            cancelAndClearTouchTargets(ev);
            //重置Touch状态标识
            resetTouchState();
        }

        // Check for interception. 检查是否拦截
        final boolean intercepted;
        //事件为ACTION_DOWN或者mFirstTouchTarget不为null(即已经找到能够接收touch事件的目标组件)时if成立
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {//如果没有设置不允许拦截 调用onInterceptTouchEvent 获取是否拦截点击事件
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // restore action in case it was changed
            } else {
                intercepted = false;
            }
        } else {
            // There are no touch targets and this action is not an initial down
            // so this view group continues to intercept touches.
            intercepted = true;
        }

        // If intercepted, start normal event dispatch. Also if there is already
        // a view that is handling the gesture, do normal event dispatch.
        if (intercepted || mFirstTouchTarget != null) {
            ev.setTargetAccessibilityFocus(false);
        }

        // Check for cancelation. 检查cancel
        final boolean canceled = resetCancelNextUpFlag(this)
                || actionMasked == MotionEvent.ACTION_CANCEL;

        // Update list of touch targets for pointer down, if needed.
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
        //不是ACTION_CANCEL并且不拦截 一般来说就是 onInterceptTouchEvent() 返回false
        if (!canceled && !intercepted) {

            // If the event is targeting accessiiblity focus we give it to the
            // view that has accessibility focus and if it does not handle it
            // we clear the flag and dispatch the event to all children as usual.
            // We are looking up the accessibility focused host to avoid keeping
            // state since these events are very rare.
            View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                    ? findChildWithAccessibilityFocus() : null;
            //开始处理dow事件
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                final int actionIndex = ev.getActionIndex(); // always 0 for down
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                        : TouchTarget.ALL_POINTER_IDS;

                // Clean up earlier touch targets for this pointer id in case they
                // have become out of sync.
                removePointersFromTouchTargets(idBitsToAssign);

                final int childrenCount = mChildrenCount;
                if (newTouchTarget == null && childrenCount != 0) {
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    // Find a child that can receive the event.
                    // Scan children from front to back.
                    //从前往后扫描查找可以接收事件的 ChlidView
                    final ArrayList<View> preorderedList = buildOrderedChildList();
                    final boolean customOrder = preorderedList == null
                            && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        final int childIndex = customOrder
                                ? getChildDrawingOrder(childrenCount, i) : i;
                        final View child = (preorderedList == null)
                                ? children[childIndex] : preorderedList.get(childIndex);

                        // If there is a view that has accessibility focus we want it
                        // to get the event first and if not handled we will perform a
                        // normal dispatch. We may do a double iteration but this is
                        // safer given the timeframe.
                        if (childWithAccessibilityFocus != null) {
                            if (childWithAccessibilityFocus != child) {
                                continue;
                            }
                            childWithAccessibilityFocus = null;
                            i = childrenCount - 1;
                        }

                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }

                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                             //找到点击范围内的childView 跳出循环
                            // Child is already receiving touch within its bounds.
                            // Give it the new pointer in addition to the ones it is handling.
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }
                        /**
                         * 如果上面的if不满足,当然也不会执行break语句.
                         * 于是代码会执行到这里来.
                         *
                         * 调用方法dispatchTransformedTouchEvent()将Touch事件传递给子View做
                         * 递归处理(也就是遍历该子View的View树)
                         * 该方法很重要,看一下源码中关于该方法的描述:
                         * Transforms a motion event into the coordinate space of a particular child view,
                         * filters out irrelevant pointer ids, and overrides its action if necessary.
                         * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
                         * 将Touch事件传递给特定的子View.
                         * 该方法十分重要!!!!在该方法中为一个递归调用,会递归调用dispatchTouchEvent()方法！！！！！！！！！！！！！！
                         * 在dispatchTouchEvent()中:
                         *当child == null时会将Touch事件传递给该ViewGroup自身的dispatchTouchEvent()处理.
                         * 如果子View为ViewGroup并且Touch没有被拦截那么递归调用dispatchTouchEvent()
                         * 如果子View为View那么就会调用其onTouchEvent(),这个就不再赘述了.
                         *
                         *
                         * 该方法返回true则表示子View消费掉该事件,同时进入该if判断.
                         * 满足if语句后重要的操作有:
                         * 1 给newTouchTarget赋值
                         * 2 给alreadyDispatchedToNewTouchTarget赋值为true.
                         *   看这个比较长的英语名字也可知其含义:已经将Touch派发给新的TouchTarget
                         * 3 执行break.
                         *   因为该for循环遍历子View判断哪个子View接受Touch事件,既然已经找到了
                         *   那么就跳出该for循环.
                         * 4 注意:
                         *   如果dispatchTransformedTouchEvent()返回false即子View
                         *   的onTouchEvent返回false(即Touch事件未被消费)那么就不满足该if条件,也就无法执行addTouchTarget()
                         *   从而导致mFirstTouchTarget为null.那么该子View就无法继续处理ACTION_MOVE事件
                         *   和ACTION_UP事件
                         * 5 注意:
                         *   如果dispatchTransformedTouchEvent()返回true即子View
                         *   的onTouchEvent返回true(即Touch事件被消费)那么就满足该if条件.
                         *   从而mFirstTouchTarget不为null
                         * 6 小结:
                         *   对于此处ACTION_DOWN的处理具体体现在dispatchTransformedTouchEvent()
                         *   该方法返回boolean,如下:
                         *   true---->事件被消费----->mFirstTouchTarget!=null
                         *   false--->事件未被消费---->mFirstTouchTarget==null
                         *   因为在dispatchTransformedTouchEvent()会调用递归调用dispatchTouchEvent()和onTouchEvent()
                         *   所以dispatchTransformedTouchEvent()的返回值实际上是由onTouchEvent()决定的.
                         *   简单地说onTouchEvent()是否消费了Touch事件(true or false)的返回值决定了dispatchTransformedTouchEvent()
                         *   的返回值!!!!!!!!!!!!!从而决定了mFirstTouchTarget是否为null!!!!!!!!!!!!!!!!从而进一步决定了ViewGroup是否
                         *   处理Touch事件.这一点在下面的代码中很有体现.
                         *   
                         *
                         */
                        resetCancelNextUpFlag(child);
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            // Child wants to receive touch within its bounds.
                            mLastTouchDownTime = ev.getDownTime();
                            if (preorderedList != null) {
                                // childIndex points into presorted list, find original index
                                for (int j = 0; j < childrenCount; j++) {
                                    if (children[childIndex] == mChildren[j]) {
                                        mLastTouchDownIndex = j;
                                        break;
                                    }
                                }
                            } else {
                                mLastTouchDownIndex = childIndex;
                            }
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }

                        // The accessibility focus didn't handle the event, so clear
                        // the flag and do a normal dispatch to all children.
                        ev.setTargetAccessibilityFocus(false);
                    }
                    if (preorderedList != null) preorderedList.clear();
                }


                /**
                * 该if条件表示:
                * 经过前面的for循环没有找到子View接收Touch事件并且之前的mFirstTouchTarget不为空
                */
                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    // Did not find a child to receive the event.
                    // Assign the pointer to the least recently added target.
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    //newTouchTarget指向了最初的TouchTarget
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
            }
        }
        /**
        * 分发Touch事件至target(Dispatch to touch targets)
        *
        * 经过上面对于ACTION_DOWN的处理后mFirstTouchTarget有两种情况:
        * 1 mFirstTouchTarget为null
        * 2 mFirstTouchTarget不为null
        *
        * 当然如果不是ACTION_DOWN就不会经过上面较繁琐的流程
        * 而是从此处开始执行,比如ACTION_MOVE和ACTION_UP
        */
        // Dispatch to touch targets.
        if (mFirstTouchTarget == null) {
          /**
            	 * 情况1：mFirstTouchTarget为null
            	 *
            	 * 经过上面的分析mFirstTouchTarget为null就是说Touch事件未被消费.
            	 * 即没有找到能够消费touch事件的子组件或Touch事件被拦截了，
            	 * 则调用ViewGroup的dispatchTransformedTouchEvent()方法处理Touch事件则和普通View一样.
            	 * 即子View没有消费Touch事件,那么子View的上层ViewGroup才会调用其onTouchEvent()处理Touch事件.
            	 * 在源码中的注释为:No touch targets so treat this as an ordinary view.
            	 * 也就是说此时ViewGroup像一个普通的View那样调用dispatchTouchEvent(),且在dispatchTouchEvent()
            	 * 中会去调用onTouchEvent()方法.
            	 * 具体的说就是在调用dispatchTransformedTouchEvent()时第三个参数为null.
            	 * 第三个参数View child为null会做什么样的处理呢?
            	 * 请参见下面dispatchTransformedTouchEvent()的源码分析
            	 *
            	 * 这就是为什么子view对于Touch事件处理返回true那么其上层的ViewGroup就无法处理Touch事件了!!!!!!!!!!
            	 * 这就是为什么子view对于Touch事件处理返回false那么其上层的ViewGroup才可以处理Touch事件!!!!!!!!!!
            	 *
            	 */
            // No touch targets so treat this as an ordinary view.
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
             //情况2：mFirstTouchTarget不为null即找到了可以消费Touch事件的子View且后续Touch事件可以传递到该子View
            // Dispatch to touch targets, excluding the new touch target if we already
            // dispatched to it.  Cancel touch targets if necessary.
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
                    //对于非ACTION_DOWN事件继续传递给目标子组件进行处理,依然是递归调用dispatchTransformedTouchEvent()        
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        target.recycle();
                        target = next;
                        continue;
                    }
                }
                predecessor = target;
                target = next;
            }
        }
        //处理ACTION_UP和ACTION_CANCEL 在此主要的操作是还原状态
        // Update list of touch targets for pointer up or cancel, if needed.
        if (canceled
                || actionMasked == MotionEvent.ACTION_UP
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            removePointersFromTouchTargets(idBitsToRemove);
        }
    }

    if (!handled && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
    }
    return handled;
}
```
dispatchTransformedTouchEvent 方法
```java
/**
 * Transforms a motion event into the coordinate space of a particular child view,
 * filters out irrelevant pointer ids, and overrides its action if necessary.
 * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
 */
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
        View child, int desiredPointerIdBits) {
    final boolean handled;

    // Canceling motions is a special case.  We don't need to perform any transformations
    // or filtering.  The important part is the action, not the contents.
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            handled = super.dispatchTouchEvent(event);//child为空时说明事件只发生在ViewGroup上 不在子View的范围内 此时调用的是父类View的dispatchTouchEvent
        } else {
            handled = child.dispatchTouchEvent(event);//直接调用子类的dispatchTouchEvent
        }
        event.setAction(oldAction);
        return handled;
    }

    // Calculate the number of pointers to deliver.
    final int oldPointerIdBits = event.getPointerIdBits();
    final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

    // If for some reason we ended up in an inconsistent state where it looks like we
    // might produce a motion event with no pointers in it, then drop the event.
    if (newPointerIdBits == 0) {
        return false;
    }

    // If the number of pointers is the same and we don't need to perform any fancy
    // irreversible transformations, then we can reuse the motion event for this
    // dispatch as long as we are careful to revert any changes we make.
    // Otherwise we need to make a copy.
    final MotionEvent transformedEvent;
    if (newPointerIdBits == oldPointerIdBits) {
        if (child == null || child.hasIdentityMatrix()) {
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                final float offsetX = mScrollX - child.mLeft;
                final float offsetY = mScrollY - child.mTop;
                event.offsetLocation(offsetX, offsetY);

                handled = child.dispatchTouchEvent(event);

                event.offsetLocation(-offsetX, -offsetY);
            }
            return handled;
        }
        transformedEvent = MotionEvent.obtain(event);
    } else {
        transformedEvent = event.split(newPointerIdBits);
    }

    // Perform any necessary transformations and dispatch.
    if (child == null) {
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
        if (! child.hasIdentityMatrix()) {
            transformedEvent.transform(child.getInverseMatrix());
        }

        handled = child.dispatchTouchEvent(transformedEvent);
    }

    // Done.
    transformedEvent.recycle();
    return handled;
}
```

View 的dispatchTouchEvent()方法
```java
/**
    * Pass the touch screen motion event down to the target view, or this
    * view if it is the target.
    *
    * @param event The motion event to be dispatched.
    * @return True if the event was handled by the view, false otherwise.
    */
   public boolean dispatchTouchEvent(MotionEvent event) {
       // If the event should be handled by accessibility focus first.
       if (event.isTargetAccessibilityFocus()) {
           // We don't have focus or no virtual descendant has it, do not handle the event.
           if (!isAccessibilityFocusedViewOrHost()) {
               return false;
           }
           // We have focus and got the event, then use normal event dispatch.
           event.setTargetAccessibilityFocus(false);
       }

       boolean result = false;

       if (mInputEventConsistencyVerifier != null) {
           mInputEventConsistencyVerifier.onTouchEvent(event, 0);
       }

       final int actionMasked = event.getActionMasked();
       if (actionMasked == MotionEvent.ACTION_DOWN) {
           // Defensive cleanup for new gesture
           stopNestedScroll();
       }

       if (onFilterTouchEventForSecurity(event)) {
           //noinspection SimplifiableIfStatement
           ListenerInfo li = mListenerInfo;
           //主要逻辑在以下代码
           //判断是否有TouchListener 执行后直接返回ture 表示事件已经被View处理 这也是解释为什么给view设置了TouchListener后不会执行 onTouchEvent()方法的原因
           if (li != null && li.mOnTouchListener != null
                   && (mViewFlags & ENABLED_MASK) == ENABLED
                   && li.mOnTouchListener.onTouch(this, event)) {
               result = true;
           }

           if (!result && onTouchEvent(event)) {//如果onTouchEvent返回的不是true 则不能返回true 会影响dispatchTransformedTouchEvent方法的返回值 那么mFirstTouchTarget==null条件成立后 ViewGroup并不会处理任何除了cancel的其他的touch事件了
               result = true;
           }
       }

       if (!result && mInputEventConsistencyVerifier != null) {
           mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
       }

       // Clean up after nested scrolls if this is the end of a gesture;
       // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
       // of the gesture.
       if (actionMasked == MotionEvent.ACTION_UP ||
               actionMasked == MotionEvent.ACTION_CANCEL ||
               (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
           stopNestedScroll();
       }

       return result;
   }
```
