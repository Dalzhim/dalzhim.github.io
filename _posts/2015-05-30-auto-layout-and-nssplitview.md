---
tags: objc
title: Auto Layout and NSSplitView
---
Auto layout is an incredibly powerful tool and the official documentation advertises it as something that is easy and natural to use. Yet, there are some dark corners that aren't explored at all within it. Among those, the `NSSplitView` is one that has been quite a piece of work for me. This article is meant to shed some light on how to use auto layout with Apple's `NSSplitView`.

The `NSSplitViewDelegate` protocol offers a way to constrain the way the splitters can be moved in order to make it possible to implement minimum and maximum sizes for every pane within the `NSSplitView`. Those methods are `-[NSSplitViewDelegate splitView:constrainMaxCoordinate:ofSubviewAt:]`, `-[NSSplitViewDelegate splitView:constrainMinCoordinate:ofSubviewAt:]` and `-[NSSplitViewDelegate splitView:constrainSplitPosition:ofSubviewAt:]`. Out of those delegate methods, `splitView:constrainMaxCoordinate:ofSubviewAt:` and `splitView:constrainMinCoordinate:ofSubviewAt:` are incompatible with auto layout. This is particularly painful when trying to convert an existing application to auto layout. Even though Apple advertises that auto layout can be adopted incrementally, you cannot use a single constraint inside a window where there is a `NSSplitView` that relies on one of those two incompatible methods without everything going wrong.

In order to resolve this issue, constraints should be added to the different panes in order to set minimum and maximum sizes. Let's assume that we have a vertical `NSSplitView` (panes are arrayed horizontally), then we need to constrain the `NSLayoutAttributeWidth` using `NSLayoutRelationGreaterThanOrEqual` and `NSLayoutRelationLessThanOrEqual` to the desired minimum and maximum respectively. As an example, here is a pane that is >= 200 and smaller than a third of the width of its superview.

```objc
[leftPane addConstraint:[NSLayoutConstraint constraintWithItem:leftPane attribute:NSLayoutAttributeWidth relatedBy:NSLayoutRelationGreaterThanOrEqual toItem:nil attribute:NSLayoutAttributeNotAnAttribute multiplier:1 constant:200]];
[parentView addConstraint:[NSLayoutConstraint constraintWithItem:leftPane attribute:NSLayoutAttributeWidth relatedBy:NSLayoutRelationGreaterThanOrEqual toItem:parentView attribute:NSLayoutAttributeWidth multiplier:1./3. constant:0]];
```

All of this works pretty easily when using a simple setup with two panes. But it gets more complicated when there can be two nested `NSSplitView` instances. Defining minimum width constraints on the three panes of two nested `NSSplitView` instances leads to a situation where dragging the splitter at index 1 may affect the position of the splitter at index 0. This happens because `NSSplitView` adds a temporary constraint while dragging a splitter. This constraint makes the `NSLayoutAttributeRight` of the pane on the left side of the splitter equal to the `NSLayoutAttributeLeft` of the `NSSplitView` itself with a constant equal to the position where the cursor is dragging the splitter. As a consequence the fittingSize of the `NSSplitView` may increase up to the point that it gets larger than its current size which in turns displaces the splitter of the parent `NSSplitView`.

The temporary constraint that makes it possible to drag a splitter lives only as long as the -[NSSplitView mouseDown:] method is executing. The reason is that a nested `NSRunLoop` is being run in `NSEventTrackingRunLoopMode` to drag the splitter until a mouseUp: event is consumed and the mouseDown: method returns. In order to prevent a splitter being relocated while dragging another splitter, we need to make sure the innermost `NSSplitView` does not grow outwardly. This is done by subclassing `NSSplitView` and overriding the `mouseDown:` method the following way :
```objc
@interface GALNestableSplitView : NSSplitView

@property(strong) NSLayoutConstraint* temporaryWidthConstraint;

@end

@implementation GALNestableSplitView

- (void)mouseDown:(NSEvent *)theEvent
{
	if (!self.temporaryWidthConstraint) {
		self.temporaryWidthConstraint = [NSLayoutConstraint constraintWithItem:self attribute:NSLayoutAttributeWidth relatedBy:NSLayoutRelationEqual toItem:nil attribute:NSLayoutAttributeNotAnAttribute multiplier:1 constant:0];
	}
	self.temporaryWidthConstraint.constant = NSWidth(self.bounds);
	[self addConstraint:self.temporaryWidthConstraint];
	[super mouseDown:theEvent]; // This call is blocking until the drag is finished
	[self removeConstraint:self.temporaryWidthConstraint];
}

@end
```

