---
layout: post
title:  "preferredMaxLayoutWidth"
date:   2013-11-16 12:23:07
categories: ios
---

## The problem

We have a custom subclass of UIView with a multiline UILabel subview. Height of the view should extend dynamically to fit the label with its content. 

## The solution 

Size of a UILabel is calculated by the system using the value specified as a preffered maximal width. From the Apple's documentation on a UILabel's preferredMaxLayoutWidth:

> This property affects the size of the label when layout constraints are applied to it. During layout, if the text extends beyond the width specified by this property, the additional text is flowed to one or more new lines, thereby increasing the height of the label

But in times of Auto Layout it would be a sad thing to calculate the preferred width outside the world of constraints. Florian Kugler lends a helping hand with his article on [objc.io][objc_issue]. In short: he suggests to let the Auto Layout do its work and use the resulting frame to update the preferred maximum width.  

It turns out that there is an undocumented feature in iOS 7 that makes the calculations for us. This means that on the newest system the Florian's trick is not needed anymore! We should use it only if necessary.
   
``` objc
- (void)layoutSubviews
{
    if (floor(NSFoundationVersionNumber) <= NSFoundationVersionNumber_iOS_6_1) {
        [super layoutSubviews];
        self.label.preferredMaxLayoutWidth = self.label.bounds.size.width;
    }
    [super layoutSubviews];
}
```

For our custom view to extend together with a label we only need to set up right constraints. Label's top should stick to the top of the view and label's bottom to the bottom.

``` objc
NSNumber * padding = @(4);
NSDictionary * metrics = NSDictionaryOfVariableBindings(padding);
NSDictionary * views = NSDictionaryOfVariableBindings(_label);
[self addConstraints:[NSLayoutConstraint constraintsWithVisualFormat:@"H:|-padding-[_label]-padding-|"
                                                             options:0
                                                             metrics:metrics
                                                               views:views]];
[self addConstraints:[NSLayoutConstraint constraintsWithVisualFormat:@"V:|-padding-[_label]-padding-|"
                                                             options:0
                                                             metrics:metrics
                                                               views:views]];
``` 
Remember that the view itself shall NOT have a height constraint.

## UITableViewCell
This obviously won't work if a label is a subview of a table view cell. Cell's height is explicitly defined by the return of the UITableViewDelegate's method tableView:heightForRowAtIndexPath:. In case you'd like to calculate the cell's height based on dynamic text remember about two things (at least).  

* The final height reserved for your cell's view is the result of heightForRowAtIndexPath decreased by the height of the table view separator. In example if the heightForRowAtIndexPath=40 and tableView.separatorStyle is set to UITableViewCellSeparatorStyleSingleLine then the cell's height=39.5 

* NSString instance method sizeWithFont:constrainedToSize: returns an exact size on iOS 7, while on iOS 6 the result is rounded. It might have an impact on your layout. 

Demo project on [github][demo_project].

[objc_issue]: http://www.objc.io/issue-3/advanced-auto-layout-toolbox.html
[demo_project]: https://github.com/danielgarbien/PreferredMaxLayoutWidthDemo
