　###

**想必大家已经对互联网传统的照片布局方式司空见惯了，这种行列分明的布局虽然对用户来说简洁明了，但是长久的使用难免会产生审美疲劳。现在网上流行一种叫做“瀑布流”的照片布局样式，这种行与列参差不齐的状态着实给用户眼前一亮的感觉**

###效果图

![瀑布流](http://upload-images.jianshu.io/upload_images/1418424-711c364cc2f21034.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###实现瀑布流的三种方式

* 第一种:(最原始，最为麻烦) 使用一个UIScrollView上面放着三个UITabelView然后在禁止UITabelView的滑动，只允许背后的UIScrollView滑动
* 第二种:(也比较麻烦)只是用UIScrollView，但是比较多的时候 我们也是需要重用，这时我们需要自己 设计出一套重用机制。
* 第三种:(最简单 UICollectionViewLayout) 在UICollectionView出来之后 我们通常都是使用这种方式来实现瀑布流，因为我们不需要设计重用机制。


`我今天我们讲的就是第三种方式来实现瀑布流`
**实现思路：**  就是每一个cell都会加到最短的高度上

***


* UICollectionViewLayout的实现 
我们先创建一个继承于`UICollectionViewLayout`  WaterLayout

***
 * .h 文件

```
#import <UIKit/UIKit.h>
//前置声明
@class XMGWaterflowLayout;
//代理
@protocol XMGWaterflowLayoutDelegate <NSObject>

@required
//必须实现的代理方法
- (CGFloat)waterflowLayout:(XMGWaterflowLayout *)waterflowLayout heightForItemAtIndex:(NSUInteger)index itemWidth:(CGFloat)itemWidth;
//可实现可不实现的代理方法 这个是通过代理调用的接口  也可以 通过set方法来实现接口
@optional
- (CGFloat)columnCountInWaterflowLayout:(XMGWaterflowLayout *)waterflowLayout;
- (CGFloat)columnMarginInWaterflowLayout:(XMGWaterflowLayout *)waterflowLayout;
- (CGFloat)rowMarginInWaterflowLayout:(XMGWaterflowLayout *)waterflowLayout;
- (UIEdgeInsets)edgeInsetsInWaterflowLayout:(XMGWaterflowLayout *)waterflowLayout;
@end

@interface XMGWaterflowLayout : UICollectionViewLayout
/** 代理 */
@property (nonatomic, weak) id<XMGWaterflowLayoutDelegate> delegate;
@end
```

*  在WaterLayout的.m文件
```
/** 默认的列数 */
static const NSInteger XMGDefaultColumnCount = 3;
/** 每一列之间的间距 */
static const CGFloat XMGDefaultColumnMargin = 10;
/** 每一行之间的间距 */
static const CGFloat XMGDefaultRowMargin = 10;
/** 边缘间距 */
static const UIEdgeInsets XMGDefaultEdgeInsets = {10, 10, 10, 10};
```

* 在.m的@interface classion中

```
/** 存放所有cell的布局属性 */
@property (nonatomic, strong) NSMutableArray *attrsArray;
/** 存放所有列的当前高度 */
@property (nonatomic, strong) NSMutableArray *columnHeights;
/** 内容的高度 */
@property (nonatomic, assign) CGFloat contentHeight;
//这里方法为了简化 数据处理(通过getter方法来实现)
- (CGFloat)rowMargin;
- (CGFloat)columnMargin;
- (NSInteger)columnCount;
- (UIEdgeInsets)edgeInsets;
```

* 这些getter方法的实现 其他的同理

```
#pragma mark - 常见数据处理
- (CGFloat)rowMargin
{
//如果delegate方法没有实现的话就使用代理方法 否则就使用默认值 然后通过getter方法来判断 来简化代码
    if ([self.delegate respondsToSelector:@selector(rowMarginInWaterflowLayout:)]) {
        return [self.delegate rowMarginInWaterflowLayout:self];
    } else {
        return XMGDefaultRowMargin;
    }
}
```

* .m文件中的瀑布流主要的实现

```
/**
 * 初始化
 */
- (void)prepareLayout
{
     [super prepareLayout];
    //先初始化内容的高度为0
     self.contentHeight = 0;
    // 清除以前计算的所有高度
    [self.columnHeights removeAllObjects];
    //先初始化  存放所有列的当前高度 3个值
    for (NSInteger i = 0; i < self.columnCount; i++) {
        [self.columnHeights addObject:@(self.edgeInsets.top)];
    }

    // 清除之前所有的布局属性
    [self.attrsArray removeAllObjects];
    // 开始创建每一个cell对应的布局属性
    NSInteger count = [self.collectionView numberOfItemsInSection:0];
    for (NSInteger i = 0; i < count; i++) {
        // 创建位置
        NSIndexPath *indexPath = [NSIndexPath indexPathForItem:i inSection:0];
        // 获取indexPath位置cell对应的布局属性
        UICollectionViewLayoutAttributes *attrs = [self layoutAttributesForItemAtIndexPath:indexPath];
        [self.attrsArray addObject:attrs];
    }
}

/**
 * 决定cell的排布
 */
- (NSArray *)layoutAttributesForElementsInRect:(CGRect)rect
{
    return self.attrsArray;
}

/**
 * 返回indexPath位置cell对应的布局属性
 */
- (UICollectionViewLayoutAttributes *)layoutAttributesForItemAtIndexPath:(NSIndexPath *)indexPath
{
    // 创建布局属性
    UICollectionViewLayoutAttributes *attrs = [UICollectionViewLayoutAttributes layoutAttributesForCellWithIndexPath:indexPath];
    
    // collectionView的宽度
    CGFloat collectionViewW = self.collectionView.frame.size.width;
    
    // 设置布局属性的frame
    CGFloat w = (collectionViewW - self.edgeInsets.left - self.edgeInsets.right - (self.columnCount - 1) * self.columnMargin) / self.columnCount;
    CGFloat h = [self.delegate waterflowLayout:self heightForItemAtIndex:indexPath.item itemWidth:w];
    
    // 找出高度最短的那一列
    //找出来最短后 就把下一个cell 添加到低下
    NSInteger destColumn = 0;
    CGFloat minColumnHeight = [self.columnHeights[0] doubleValue];
    for (NSInteger i = 1; i < self.columnCount; i++) {
        // 取得第i列的高度
        CGFloat columnHeight = [self.columnHeights[i] doubleValue];
        
        if (minColumnHeight > columnHeight) {
            minColumnHeight = columnHeight;
            destColumn = i;
        }
    }
    
    CGFloat x = self.edgeInsets.left + destColumn * (w + self.columnMargin);
    CGFloat y = minColumnHeight;
    if (y != self.edgeInsets.top) {
        y += self.rowMargin;
    }
    attrs.frame = CGRectMake(x, y, w, h);
    
    // 更新最短那列的高度
    self.columnHeights[destColumn] = @(CGRectGetMaxY(attrs.frame));
    
    // 记录内容的高度
    CGFloat columnHeight = [self.columnHeights[destColumn] doubleValue];
    //找出最高的高度
    if (self.contentHeight < columnHeight) {
        self.contentHeight = columnHeight;
    }
    return attrs;
}

/*!
 *  返回值就是这个CollectionView的contensize 因为CollectionView也是ScrollView的子类
 */
- (CGSize)collectionViewContentSize
{
//    CGFloat maxColumnHeight = [self.columnHeights[0] doubleValue];
//    for (NSInteger i = 1; i < self.columnCount; i++) {
//        // 取得第i列的高度
//        CGFloat columnHeight = [self.columnHeights[i] doubleValue];
//        
//        if (maxColumnHeight < columnHeight) {
//            maxColumnHeight = columnHeight;
//        }
//    }
    return CGSizeMake(0, self.contentHeight + self.edgeInsets.bottom);
}

```

* 在ViewController中的使用 

```
 // 创建布局
    XMGWaterflowLayout *layout = [[XMGWaterflowLayout alloc] init];
    layout.delegate = self;
//然后继承代理实现代理方法

#pragma mark - <WaterLayoutDelegate>
//返回每个cell的高度
- (CGFloat)waterflowLayout:(XMGWaterflowLayout *)waterflowLayout heightForItemAtIndex:(NSUInteger)index itemWidth:(CGFloat)itemWidth
{
    XMGShop *shop = self.shops[index];
    return itemWidth * shop.h / shop.w;
}

//每行的最小距离
- (CGFloat)rowMarginInWaterflowLayout:(XMGWaterflowLayout *)waterflowLayout
{
    return 10;
}
//有多少列
- (CGFloat)columnCountInWaterflowLayout:(XMGWaterflowLayout *)waterflowLayout
{
    if (self.shops.count <= 50) return 2;
    return 3;
}

//内边距
- (UIEdgeInsets)edgeInsetsInWaterflowLayout:(XMGWaterflowLayout *)waterflowLayout
{
    return UIEdgeInsetsMake(10, 20, 30, 100);
}
```
[UIcollectionViewLayout的布局详解](http://www.jianshu.com/p/45ff718090a8)
