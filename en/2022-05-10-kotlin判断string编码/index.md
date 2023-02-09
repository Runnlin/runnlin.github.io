# Kotlin判断String编码




# Koting String 编码判断
  
  > 在项目中，需要支持UTF16或其他编码格式的显示String，如果直接按照默认的UTF8来输出的话，会导致显示乱码
  
  判断方法：
  ``` Java
  /*
  转换特殊编码String
   */
  private fun String.toUTF8String(): String {
    // 内联函数
    val ins = intern().byteInputStream()
    val head = ByteArray(3)
    ins.read(head)
    Log.i(TAG, "code:${head[0]},${head[1]}")
    return when {
        // 这是UTF16,转换后会产生两个字节的乱码字符，需要进行裁剪
        head[0].toInt() == -28 && head[1].toInt() == -92 -> String(
            intern().toByteArray(Charsets.UTF_16),
            Charset.defaultCharset()
        ).substring(2)
        // unicode
        head[0].toInt() == -2 && head[1].toInt() == -1 -> intern()
        // UTF8
        head[0].toInt() == -17 && head[1].toInt() == -69 && head[2].toInt() == -65 -> intern()
        else -> intern()
    }
  }
  ```
