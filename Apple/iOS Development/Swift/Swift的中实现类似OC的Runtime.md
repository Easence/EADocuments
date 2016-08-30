# Swift的中实现类似OC的Runtime

标签（空格分隔）： 开发 Swift

---
在OC上我们能利用OC的Runtime来改变对象的方法实现、添加方法、改变类型等。Swift是门静态语言，在编译器已经决定了方法的调用地址，那在Swift中是否也有这样的特性呢？

- NSObject的子类具有Runtime特性，编译器会给她们加上@objc关键字。
- 纯Swift对象默认是没有Runtime特性的，以及Swift特有的类型（如：Tupe、Character、String、AnyObject）也是没有Runtime特性。
- 手动在属性、方法前加上@objc关键字或许可以实现Runtime特性，编译器并不保证能100%成功。
- **正确的做法应该是在属性、方法前加上dynamic关键字，这样编译器就会在相应的属性、方法前加@objc，可以实现完全动态化。**





