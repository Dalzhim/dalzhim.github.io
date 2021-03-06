---
tags: objc
title: Auto Layout and NSScrollView
---
Apple's auto layout guide states that constraints cannot be set to cross a view within the hierarchy if it sets the bounds manually using a `-[NSView layout]` or `-[UIView layoutSubviews]` override. It is also not possible to cross a view that has a bounds transform such as the `NSScrollView`. What this means, is that because the constraints aren't aware of the bounds transform or the manual manipulations on the frame property of a view, a hierarchy of views where the ancestor A containing a `NSScrollView` S which in turn contains a child C cannot set constraints on C that are related to A. S acts as a barrier for the constraint system.

Those who have tried inserting content that depends on auto layout as the document view of a `NSScrollView` may have already found the documentation to be lacking in this regard. There are usually two major problems :
1. Content that is shorter than the visible rect is aligned to the bottom edge of the `NSScrollView` and,
1. The content size does not adjust properly

<p>In order to solve the first problem, we need a custom NSView subclass that will replace the default contentView property of the NSScrollView instance. The trick here is that this subview needs to flip the coordinate system.</p>
```objc
@interface GALFlippedClipView : NSClipView
@end

@implementation GALFlippedClipView

- (BOOL)isFlipped
{
    return YES;
}

@end
```

<p>Then, this subclass needs to be installed with constraints binding it to the four edges of its parent.</p>
```objc
NSScrollView* scrollView = [[NSScrollView alloc] init];
NSView* contentView  = [[GALFlippedClipView alloc] init];
contentView.translatesAutoresizingMaskIntoConstraints = NO;
[scrollView setContentView:contentView];
[scrollView addConstraint:[NSLayoutConstraint constraintWithItem:contentView attribute:NSLayoutAttributeLeft relatedBy:NSLayoutRelationEqual toItem:scrollView attribute:NSLayoutAttributeLeft multiplier:1. constant:0.]];
[scrollView addConstraint:[NSLayoutConstraint constraintWithItem:contentView attribute:NSLayoutAttributeTop relatedBy:NSLayoutRelationEqual toItem:scrollView attribute:NSLayoutAttributeTop multiplier:1. constant:0.]];
[scrollView addConstraint:[NSLayoutConstraint constraintWithItem:contentView attribute:NSLayoutAttributeRight relatedBy:NSLayoutRelationEqual toItem:scrollView attribute:NSLayoutAttributeRight multiplier:1. constant:0.]];
[scrollView addConstraint:[NSLayoutConstraint constraintWithItem:contentView attribute:NSLayoutAttributeBottom relatedBy:NSLayoutRelationEqual toItem:scrollView attribute:NSLayoutAttributeBottom multiplier:1. constant:0.]];
```

<p>With this setup, it is now possible to insert a documentView that uses auto layout by binding it to the contentView's left, top and right edges the following way.</p>
```objc
NSView* documentView = /* … */;
documentView.translatesAutoresizingMaskIntoConstraints = NO;
[scrollView setDocumentView:documentView];
[contentView addConstraint:[NSLayoutConstraint constraintWithItem:documentView attribute:NSLayoutAttributeLeft relatedBy:NSLayoutRelationEqual toItem:contentView attribute:NSLayoutAttributeLeft multiplier:1. constant:0.]];
[contentView addConstraint:[NSLayoutConstraint constraintWithItem:documentView attribute:NSLayoutAttributeTop relatedBy:NSLayoutRelationEqual toItem:contentView attribute:NSLayoutAttributeTop multiplier:1. constant:0.]];
[contentView addConstraint:[NSLayoutConstraint constraintWithItem:documentView attribute:NSLayoutAttributeRight relatedBy:NSLayoutRelationEqual toItem:contentView attribute:NSLayoutAttributeRight multiplier:1. constant:0.]];
```

<p>In order for this setup to work properly, the documentView needs to have a valid fittingSize. The fittingSize is documented as the minimum size that will allow the views to be displayed properly. What's interesting here is that there are no subclasses and overrides required to generate content that has a proper fittingSize. This property is inferred from constraints such as :</p>
<ul>
<li>horizontal/vertical space between siblings,</li>
<li>margins between parent view and subview,</li>
<li>height and width constraints,</li>
<li>the intrinsicContentSize property,</li>
<li>the fittingSize property when the intrinsicContentSize property has no intrinsic metric for a particular orientation.</li>
</ul>
<p>Let's consider a NSView P which contains a label L and a textField T. We could create constraints with the visual format string H:|-20-[L]-10-[T(80)]-20-| and the layout would be computed the following way :</p>
<ol>
<li>P's intrinsicContentSize property has no intrinsic metric for both layout orientations so there are no Width or Height constraints created automatically,</li>
<li>P does not have any other Width or Height constraints installed programmatically,</li>
<li>P's fittingSize is computed the following way :
<ol>
<li>The first constraint held by P is a margin that relates it to L with the constant 20. 20 is now our minimum width,</li>
<li>L has an intrinsicContentSize property because it is a label (a NSTextField displayed as an input field wouldn't have an intrinsic width, but a label does), let's assume the intrinsic width is 100, this means that a Width constraint has been automatically generated by the updateConstraints mechanism with the value 100, this takes the minimum total width to 120,</li>
<li>Next, we have a horizontal space of 10 between L1 and L2, making the cumulative fitting width 130,</li>
<li>Then we have a textField for which a Width constraint has been created with the visual format language with a value of 80, this makes the fitting width 210,</li>
<li>Finally, we have another margin which relates the textField to P with a 20 points constant, the end result is now 230.</li>
</ol>
</li>
<li>Applying the same computation for the vertical orientation would allow us to complete the fittingSize values for our example, but the concept remains the same. FittingSize can be computed as long as views have intrinsic content size or fixed dimensions and are related together by spacers and margins.</li>
</ol>
