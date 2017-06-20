/**
 * description:
 *
 * @auther cuihai
 * @since 2017/5/31.
 */

public class PopUpAndDownView {
    private PopupWindow popupWindow;
    private Window mWindow;
    private Context context;
    private View view;

    public PopUpAndDownView(Context context, int layoutId) {
        //View view = View.inflate(context, layoutId, null);
        view = LayoutInflater.from(context).inflate(layoutId, null);
        popupWindow = new PopupWindow(view, ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT);
        popupWindow.setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE);
        setOutsideTouchable(true);
    }

    public View getView(int Id) {
        View viewById = view.findViewById(Id);
        return viewById;
    }

    // 在parent下面显示
    public void showAsDropDown(View parent, int xoff, int yoff) {
        popupWindow.showAsDropDown(parent, xoff, yoff);
    }

    // 在某处位置显示
    public void showAtLocation(View parent, int xoff, int yoff, int gravity) {
        popupWindow.showAtLocation(parent, gravity, xoff, yoff);

    }

    public void setAnimationStyle(int animationStyle) {
        popupWindow.setAnimationStyle(animationStyle);
    }

    void setBackGroundLevel(float level) {
        mWindow = ((Activity) context).getWindow();
        WindowManager.LayoutParams params = mWindow.getAttributes();
        params.alpha = level;
        mWindow.setAttributes(params);
    }

    private void setOutsideTouchable(boolean touchable) {
        popupWindow.setBackgroundDrawable(new ColorDrawable(0x00000000));//设置透明背景
        popupWindow.setOutsideTouchable(touchable);//设置outside可点击
        popupWindow.setFocusable(touchable);
    }

    public void dismiss() {
        popupWindow.dismiss();
    }

    public boolean isShowing() {
        return popupWindow.isShowing();
    }

    // 得到真正的 popupWindow
    public PopupWindow getPopupWindow() {
        return popupWindow;
    }
}
