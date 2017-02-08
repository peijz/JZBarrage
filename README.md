# JZBarrage
首先来看一下弹幕的生成过程，初始化三个弹幕轨迹，如果不足三个，创建2个或者1个轨迹，代码（BulletManager.m）如下：

- (void)start {
    if (self.tmpComments.count == 0) {
        [self.tmpComments addObjectsFromArray:self.allComments];
    }
    self.bStarted = YES;
    self.bStopAnimation = NO;
    [self initBulletCommentView];
}
/**
  *  初始化弹幕
  */
- (void)initBulletCommentView {
    //初始化三条弹幕轨迹
    NSMutableArray *arr = [NSMutableArray arrayWithArray:@[@(0), @(1), @(2)]];
    for (int i = 3; i > 0; i--) {
        NSString *comment = [self.tmpComments firstObject];
        if (comment) {
            [self.tmpComments removeObjectAtIndex:0];
            //随机生成弹道创建弹幕进行展示（弹幕的随机飞入效果）
            NSInteger index = arc4random()%arr.count;
            Trajectory trajectory = [[arr objectAtIndex:index] intValue];
            [arr removeObjectAtIndex:index];
            [self createBulletComment:comment trajectory:trajectory];
        } else {
            break;
        }
    }
}
创建弹幕view，对弹幕view的各种位置状态进行监听并做出相对应的处理。

 /**
  *  创建弹幕
  *
  *  @param comment    弹幕内容
  *  @param trajectory 弹道位置
  */
  - (void)createBulletComment:(NSString *)comment trajectory:(Trajectory)trajectory {
       if (self.bStopAnimation) {
           return;
       }
       //创建一个弹幕view
       BulletView *view = [[BulletView alloc] initWithContent:comment];
       //设置运行轨迹
       view.trajectory = trajectory;
       __weak BulletView *weakBulletView = view;
       __weak BulletManager *myself = self;
       /**
         *  弹幕view的动画过程中的回调状态
         *  Start:创建弹幕在进入屏幕之前
         *  Enter:弹幕完全进入屏幕
         *  End:弹幕飞出屏幕后  
         */             
       view.moveBlock = ^(CommentMoveStatus status) {
           if (myself.bStopAnimation) {
               return ;
           }
           switch (status) {
               case Start:
                   //弹幕开始……将view加入弹幕管理queue
                   [self.bulletQueue addObject:weakBulletView];
                   break;
               case Enter: {
                   //弹幕完全进入屏幕，判断接下来是否还有内容，如果有则在该弹道轨迹对列中创建弹幕……
                   NSString *comment = [myself nextComment];
                   if (comment) {
                       [myself createBulletComment:comment trajectory:trajectory];
                   } else {
                       //说明到了评论的结尾了
                   }
                   break;
               }
               case End: {
                   //弹幕飞出屏幕后从弹幕管理queue中删除
                   if ([myself.bulletQueue containsObject:weakBulletView]) {
                       [myself.bulletQueue removeObject:weakBulletView];
                   }
                   if (myself.bulletQueue.count == 0) {
                       //说明屏幕上已经没有弹幕评论了，循环开始
                       [myself start];
                   }
                   break;
               }
               default:
                  break;
           }
       };    
       //弹幕生成后，传到viewcontroller进行页面展示
       if (self.generateBulletBlock) {
            self.generateBulletBlock(view);
       }
  }
  - (NSString *)nextComment {
     NSString *comment = [self.tmpComments firstObject];
     if (comment) {
         [self.tmpComments removeObjectAtIndex:0];
     }
     return comment;
  }
弹幕view的动画执行，部分代码（BulletView.m）如下：

- (void)startAnimation {
    //根据定义的duration计算速度以及完全进入屏幕的时间
    CGFloat wholeWidth = CGRectGetWidth(self.frame) + mWidth + 50;
    CGFloat speed = wholeWidth/mDuration;
    CGFloat dur = (CGRectGetWidth(self.frame) + 50)/speed;
    __block CGRect frame = self.frame;
    if (self.moveBlock) {
        //弹幕开始进入屏幕
        self.moveBlock(Start);
    }
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(dur * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
       //避免重复，通过变量判断是否已经释放了资源，释放后，不在进行操作。
       //在stopAnimation中 self.bDealloc = YES;
       if (self.bDealloc) {
              return;
       }
        //dur时间后弹幕完全进入屏幕
        if (self.moveBlock) {
            self.moveBlock(Enter);
        }
    });  
    //弹幕完全离开屏幕
    [UIView animateWithDuration:mDuration delay:0 options:UIViewAnimationOptionCurveLinear animations:^{
        frame.origin.x = -CGRectGetWidth(frame);
        self.frame = frame;
    } completion:^(BOOL finished) {
        if (self.moveBlock) {
            self.moveBlock(End);
        }
        [self removeFromSuperview];
    }];
}
在viewcontroller中直接调用以下代码：

self.bulletManager = [[BulletManager alloc] init];
__weak ViewController *myself = self;
self.bulletManager.generateBulletBlock = ^(BulletView *bulletView) {
    [myself addBulletView:bulletView];
};

- (void)addBulletView:(BulletView *)bulletView {
    bulletView.frame = CGRectMake(CGRectGetWidth(self.view.frame)+50, 200 + 34 * bulletView.trajectory, CGRectGetWidth(bulletView.bounds), CGRectGetHeight(bulletView.bounds));
    [self.view addSubview:bulletView];
    [bulletView startAnimation];
}
//点击开始按钮，弹幕开始飞入屏幕
- (void)clickStart:(UIButton *)btn {
    [self.bulletManager start];
}
