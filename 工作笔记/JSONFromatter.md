## 格式打印

* open_object_section和open_array_section几乎一样，区别仅在于是否是array的标志，实现都是在print_name后打印[或{，最后再push到m_stack
* print_name首个print_name会判断**上次**open的m_stack是否为空，
  * print_comma
    * 判断**上次**的entry.size
  * 最后会将**本次**的entry.size加1

