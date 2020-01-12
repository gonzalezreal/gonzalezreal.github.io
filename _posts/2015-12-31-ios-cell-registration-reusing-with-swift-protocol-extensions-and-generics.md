---
layout: single
title: "iOS Cell Registration and Reusing with Swift Protocol Extensions and Generics"
tags:
  - generics
  - protocol extensions
  - uikit
  - swift
date: 2015-12-31 11:47:16 +0100
---
A common task when developing iOS apps is to register custom cell subclasses for both `UITableView` and `UICollectionView`. Well, that is if you don’t use Storyboards, of course.

Both `UITableView` and `UICollectionView` offer a similar API to register custom cell classes:

```swift
public func registerClass(cellClass: AnyClass?, forCellWithReuseIdentifier identifier: String)
public func registerNib(nib: UINib?, forCellWithReuseIdentifier identifier: String)
```

A widely accepted solution to handle cell registration and dequeuing is to declare a constant for the reuse identifier:

```swift
private let reuseIdentifier = "BookCell"

class BookListViewController: UIViewController, UICollectionViewDataSource {

    @IBOutlet private weak var collectionView: UICollectionView!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let nib = UINib(nibName: "BookCell", bundle: nil)
        self.collectionView.registerNib(nib, forCellWithReuseIdentifier: reuseIdentifier)
    }

    func collectionView(collectionView: UICollectionView, cellForItemAtIndexPath indexPath: NSIndexPath) -> UICollectionViewCell {
        let cell = collectionView.dequeueReusableCellWithReuseIdentifier(reuseIdentifier, forIndexPath: indexPath)
    
        if let bookCell = cell as? BookCell {
            // TODO: configure cell
        }
    
        return cell
    }
}
```

Let’s try to generalize this code and make it simpler and safe.

First of all, it would be nice to get away with declaring a constant for every reuse identifier in our app. We can just use the name of the custom cell class as a **default reuse identifier**.
We can create a **protocol for Reusable Views** and provide a default implementation constrained to `UIView` subclasses.

```swift
protocol ReusableView: AnyObject {
    static var defaultReuseIdentifier: String { get }
}

extension ReusableView where Self: UIView {
    static var defaultReuseIdentifier: String {
        return String(describing: self)
    }
}

extension UICollectionViewCell: ReusableView {
}
```

By making `UICollectionViewCell` conform to the `ReusableView` protocol, we get a unique reuse identifier per cell subclass.

```swift
let identifier = BookCell.defaultReuseIdentifier
// identifier = "BookCell"
```

Next, we can get rid of the hard-coded string we are using to load the Nib.

Let’s create a protocol for **Nib Loadable Views** and provide a default implementation using protocol extensions.

```swift
protocol NibLoadableView: AnyObject {
    static var nibName: String { get }
}

extension NibLoadableView where Self: UIView {
    static var nibName: String {
        return String(describing: self)
    }
}

extension BookCell: NibLoadableView {
}
```

By making our `BookCell` class conform to the `NibLoadableView` protocol we now have a safer way to get the Nib name.

```swift
let nibName = BookCell.nibName
// nibName = "BookCell"
```

If you use a different name for the XIB file than the one provided by Xcode, you can always override the default implementation of the nibName property.

With these two protocols in place, we can use **Swift Generics** and extend `UICollectionView` to simplify cell registration and dequeuing.

```swift
extension UICollectionView {
    func register<T: UICollectionViewCell>(_: T.Type) where T: ReusableView {
        register(T.self, forCellWithReuseIdentifier: T.defaultReuseIdentifier)
    }

    func register<T: UICollectionViewCell>(_: T.Type) where T: ReusableView, T: NibLoadableView {
        let bundle = Bundle(for: T.self)
        let nib = UINib(nibName: T.nibName, bundle: bundle)

        register(nib, forCellWithReuseIdentifier: T.defaultReuseIdentifier)
    }
    
    func dequeueReusableCell<T: UICollectionViewCell>(_: T.Type, for indexPath: IndexPath) -> T where T: ReusableView {
        guard let cell = dequeueReusableCell(withReuseIdentifier: T.defaultReuseIdentifier, for: indexPath) as? T else {
            fatalError("Could not dequeue cell with identifier '\(T.defaultReuseIdentifier)'")
        }

        return cell
    }
}
```

Notice that we created two versions of the register method, one for cell subclasses implementing just `ReusableView` and another one for cell subclasses implementing both `ReusableView` and `NibLoadableView`. This nicely decouples the view controller from the specific cell registration method.

Another nice detail is that the `dequeueReusableCell` method doesn’t need to take any reuse identifier and uses the cell subclass type for the return value.

Now the cell registration and dequeuing code looks much better :).

```swift
class BookListViewController: UIViewController, UICollectionViewDataSource {

    @IBOutlet private weak var collectionView: UICollectionView!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        self.collectionView.register(BookCell.self)
    }

    func collectionView(collectionView: UICollectionView, cellForItemAtIndexPath indexPath: NSIndexPath) -> UICollectionViewCell {
        let cell = collectionView.dequeueReusableCell(BookCell.self, for: indexPath)
        
        // TODO: configure cell
    
        return cell
    }
    ...
}
```

## Conclusion
If you are coming from Objective-C it is worth investigating powerful Swift features like Protocol Extensions and Generics to find alternate and more elegant ways to deal with Cocoa classes.

The code in this post is available [here](https://github.com/gonzalezreal/Reusable) in the form of a convenient Swift Package.
{: .notice--info}