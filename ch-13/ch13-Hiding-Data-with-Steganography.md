# 第13章：用隐写术隐藏数据

`steganography` 一词是希腊词 `steganos`（意为覆盖、隐藏或保护）和 `graphien` （意为书写）的组合。 在安全方面，`steganography`是指用于通过将数据植入其他数据（例如图像）中来混淆（或隐藏）数据的技术和程序，以便可以在未来的某个时间点提取数据。作为安全社区的一员，通过隐藏交付给目标后恢复的有效负载来研究这一常规实践。

在本章中，你将植入 Portable Network Graphics(PNG)图片的数据。首先研究PNG格式，并学习如何读取PNG数据。 然后将自己的数据植入到现存的图像中。最后，探索XOR，一种加解密植入数据的方法。