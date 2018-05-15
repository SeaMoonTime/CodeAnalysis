# ProgressHUD源码详解

> ProgressHUD是iOS开发中常用的等待控件(菊花控件)，用户网络请求等


## 主要结构

`ProgressHUD`只包含两个代码文件`ProgressHUD.h`和`ProgressHUD.m`，以及一个资源文件`ProgressHUD.bundle`

`ProgressHUD`本质上为`UIView`的子类

```objective-c
@interface ProgressHUD : UIView
```

其主要结构包含5部分：

```objective-c
@interface ProgressHUD()
{
	UIWindow *window;
	UIView *viewBackground;
	UIToolbar *toolbarHUD;
	UIActivityIndicatorView *spinner;
	UIImageView *imageView;
	UILabel *labelStatus;
}

```

以上控件之间的结构关系如下：
![image](https://raw.githubusercontent.com/SeaMoonTime/CodeAnalysis/master/ProgressHUD/images/ProgressHUD_Structure.png)



## 初始化控件

- 在`show`方法中调用

```objective-c
+ (void)show
{
	dispatch_async(dispatch_get_main_queue(), ^{
		[[self shared] hudCreate:nil image:nil spin:YES hide:NO interaction:YES];
	});
}
```

- 初始化

```objective-c

- (void)hudCreate:(NSString *)status image:(UIImage *)image spin:(BOOL)spin hide:(BOOL)hide interaction:(BOOL)interaction
{
    //初始化toolbarHUD
	if (toolbarHUD == nil)
	{
		toolbarHUD = [[UIToolbar alloc] initWithFrame:CGRectZero];
		toolbarHUD.translucent = YES;
		toolbarHUD.backgroundColor = self.hudColor;
		toolbarHUD.layer.cornerRadius = 10;
		toolbarHUD.layer.masksToBounds = YES;
		[self registerNotifications];
	}
	
	if (toolbarHUD.superview == nil)
	{
        //如果与背景视图不可交互，初始化背景视图，背景视图为toolbarHUD的父视图
		if (interaction == NO)
		{
			viewBackground = [[UIView alloc] initWithFrame:window.frame];
			viewBackground.backgroundColor = self.backgroundColor;
			[window addSubview:viewBackground];
			[viewBackground addSubview:toolbarHUD];
		}//如果与背景视图可交互，顶层window为toolbarHUD的父视图
		else [window addSubview:toolbarHUD];
	}
	//初始化spinner(菊花),并添加到toolbarHUD
	if (spinner == nil)
	{
		spinner = [[UIActivityIndicatorView alloc] initWithActivityIndicatorStyle:UIActivityIndicatorViewStyleWhiteLarge];
		spinner.color = self.spinnerColor;
		spinner.hidesWhenStopped = YES;
	}
	if (spinner.superview == nil) [toolbarHUD addSubview:spinner];
	//初始化imageView并添加到toolbarHUD
	if (imageView == nil)
	{
		imageView = [[UIImageView alloc] initWithFrame:CGRectMake(0, 0, 28, 28)];
	}
	if (imageView.superview == nil) [toolbarHUD addSubview:imageView];
	//初始化labelStatus，添加到toolbarHUD
	if (labelStatus == nil)
	{
		labelStatus = [[UILabel alloc] initWithFrame:CGRectZero];
		labelStatus.font = self.statusFont;
		labelStatus.textColor = self.statusColor;
		labelStatus.backgroundColor = [UIColor clearColor];
		labelStatus.textAlignment = NSTextAlignmentCenter;
		labelStatus.baselineAdjustment = UIBaselineAdjustmentAlignCenters;
		labelStatus.numberOfLines = 0;
	}
	if (labelStatus.superview == nil) [toolbarHUD addSubview:labelStatus];

    //文本信息添加
	labelStatus.text = status;
	labelStatus.hidden = (status == nil) ? YES : NO;
	//图片信息添加
	imageView.image = image;
	imageView.hidden = (image == nil) ? YES : NO;
	//spinner(菊花)旋转(或停止)
	if (spin) [spinner startAnimating]; else [spinner stopAnimating];
	
	[self hudSize];//尺寸设置
	[self hudPosition:nil];//位置设置
	[self hudShow];//动画显示
	//定时关闭
	if (hide) [self timedHide];
}

```


## 设置Size

```objective-c

- (void)hudSize
{
	CGRect rectLabel = CGRectZero;
	CGFloat widthHUD = 100, heightHUD = 100;//toolbarHUD默认的尺寸
	if (labelStatus.text != nil)
	{
		NSDictionary *attributes = @{NSFontAttributeName:labelStatus.font};
		NSInteger options = NSStringDrawingUsesFontLeading | NSStringDrawingTruncatesLastVisibleLine | NSStringDrawingUsesLineFragmentOrigin;
		rectLabel = [labelStatus.text boundingRectWithSize:CGSizeMake(200, 300) options:options attributes:attributes context:NULL];//计算文字所需的Rect

        //toolbarHUD对应的Rect
		widthHUD = rectLabel.size.width + 50;
		heightHUD = rectLabel.size.height + 75;

		if (widthHUD < 100) widthHUD = 100;
		if (heightHUD < 100) heightHUD = 100;

		rectLabel.origin.x = (widthHUD - rectLabel.size.width) / 2;
		rectLabel.origin.y = (heightHUD - rectLabel.size.height) / 2 + 25;
	}
	
	toolbarHUD.bounds = CGRectMake(0, 0, widthHUD, heightHUD);
	//imageView对应的位置
	CGFloat imageX = widthHUD/2;
	CGFloat imageY = (labelStatus.text == nil) ? heightHUD/2 : 36;
	imageView.center = spinner.center = CGPointMake(imageX, imageY);
	
	labelStatus.frame = rectLabel;
}

```



## 设置postion

```objective-c
- (void)hudPosition:(NSNotification *)notification
{
	CGFloat heightKeyboard = 0;
	NSTimeInterval duration = 0;
	//计算键盘高度
	if (notification != nil)
	{
		NSDictionary *info = [notification userInfo];
		CGRect keyboard = [[info valueForKey:UIKeyboardFrameEndUserInfoKey] CGRectValue];
		duration = [[info valueForKey:UIKeyboardAnimationDurationUserInfoKey] doubleValue];
		if ((notification.name == UIKeyboardWillShowNotification) || (notification.name == UIKeyboardDidShowNotification))
		{
			heightKeyboard = keyboard.size.height;
		}
	}
	else heightKeyboard = [self keyboardHeight];
	//toolbarHUD的水平中心在屏幕中间，垂直中心在除去键盘高度的中心。
	CGRect screen = [UIScreen mainScreen].bounds;
	CGPoint center = CGPointMake(screen.size.width/2, (screen.size.height-heightKeyboard)/2);
	//动态设置toolbarHUD的中心
	[UIView animateWithDuration:duration delay:0 options:UIViewAnimationOptionAllowUserInteraction animations:^{
		self->toolbarHUD.center = CGPointMake(center.x, center.y);
	} completion:nil];
	if (viewBackground != nil) viewBackground.frame = window.frame;
}

```

- 注意:

1. `toolbarHUD`的水平中心在屏幕中间，垂直中心在除去键盘高度的中心。

```objective-c
CGPoint center = CGPointMake(screen.size.width/2, (screen.size.height-heightKeyboard)/2);
```

2. `toolbarHUD`的垂直中心会根据键盘消息进行调整。

## 显示

```objective-c
- (void)hudShow
{
	if (self.alpha == 0)
	{
		self.alpha = 1;
		toolbarHUD.alpha = 0; //初始设置为完全透明，实现隐藏功能
		toolbarHUD.transform = CGAffineTransformScale(toolbarHUD.transform, 1.4, 1.4); //放大

		UIViewAnimationOptions options = UIViewAnimationOptionAllowUserInteraction | UIViewAnimationCurveEaseOut; //动画模式设置
        //动画显示
		[UIView animateWithDuration:0.15 delay:0 options:options animations:^{
			self->toolbarHUD.transform = CGAffineTransformScale(self->toolbarHUD.transform, 1/1.4, 1/1.4); //恢复正常
			self->toolbarHUD.alpha = 1; //设为不透明，显示出来
		} completion:nil];
	}
}
```

## 隐藏

```objective-c
- (void)hudHide
{
	if (self.alpha == 1)
	{
		UIViewAnimationOptions options = UIViewAnimationOptionAllowUserInteraction | UIViewAnimationCurveEaseIn; //动画模式设置
		[UIView animateWithDuration:0.15 delay:0 options:options animations:^{
			self->toolbarHUD.transform = CGAffineTransformScale(self->toolbarHUD.transform, 0.7, 0.7); //缩小为原来的0.7倍
			self->toolbarHUD.alpha = 0; //设置为透明，实现隐藏
		}
		completion:^(BOOL finished) {
			[self hudDestroy]; //删除各控件的依赖关系，再删除控件
			self.alpha = 0;
		}];
	}
}

```