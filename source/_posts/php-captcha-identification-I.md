---
title: PHP 验证码识别1
date: 2015-10-29 22:41:00
tags:
    - PHP
    - Captcha
---

> PHP简单验证码识别，识别字符大小位置固定，干扰小的验证码。

### 思路
- 预处理图像：根据颜色深浅度预处理，将一些干扰点去除，将图像转化为黑白两色。
- 分割验证码：将各个数字分割成单独一个字符的图像。
- 采集样本：将可能出现的字符采集下来，分成样本图像，图像的名称为该图像的字符。
- 对比样本：将程序分割出来的图像与样本图像上的像素点做对比，取相似度高的值。

### 预处理图像：
设定一个RGB的阀值，低于阀值的像素点替换为白色，高于阀值的替换为黑色。将图像转化为黑白。
<!-- more -->

``` php
    /**
     * 判断是否为验证码
     * @param  image $img
     * @param  int $x   
     * @param  int $y   
     * @return boolean  
     */
    public function isInterfere($img, $x, $y)
    {
        //取得图像指定点颜色的索引值
        $rgb = imagecolorat($img, $x, $y);
        //取得索引值的RGB          
        $rgbArray = imagecolorsforindex($img, $rgb);    

        //与设定的阀值对比，确定是否为验证码部分
        if ($rgbArray['red'] + $rgbArray['green'] + $rgbArray['blue'] > 100) {
            return false;
        }
        return true;
    }


    /**
     * 预处理图像
     * @return image  
     */
    public function removeBackgroud()
    {
            //根据图像的类型创建一个新的图像
            if ($this->type == 'image/png') {
                $img = imagecreatefrompng($this->image);
            } elseif ($this->type == 'image/jpeg') {
                $img = imagecreatefromjpeg($this->image);
            } elseif ($this->type == 'image/gif') {
                $img = imagecreatefromgif($this->image);
            }

            //设置替换的颜色值
            $white = imagecolorallocate($img, 255, 255, 255);
            $black = imagecolorallocate($img, 0, 0, 0);

            //遍历验证码图像的各个像素点
            for ($x = 0; $x < $this->width; ++$x) {
                for ($y = 0; $y < $this->height; ++$y) {
                    if($this->isInterfere($img, $x, $y)) {
                        imagesetpixel($img, $x, $y, $black);
                    }else {
                        imagesetpixel($img, $x, $y, $white);
                    }
                }
            }
            return $img;
    }
```

### 分割验证码
这里因为位置是固定的所以比较容易分割。网上看的资料说，现在验证码识别的难点就是分割字符这块，有些验证码比较男去分割单个字符。这里这个是最简单的分割字符。

``` php
    /**
     * 分割字符
     * @param  image $img
     * @return array      
     */
    public function splitImage($img)
    {
        $x = 10;
        $y = 6;
        $width = 8;
        $height = 10;
        $space = 1;
        $subImgs = array();

        for ($i=0; $i < 4; $i++) {
            //创建单字符的验证码子图像
            $subImgs[$i] = imagecreatetruecolor($width, $height);

            //复制验证码上的指定字符到新建图像
            imagecopy($subImgs[$i], $img, 0, 0, $x, $y, $width, $height);
            $x += ($width + $space);
        }

        return $subImgs;
    }
```

### 采集样本
这里需要采集到所有可能出现的字符，对比起英文验证码，中文验证码更难识别。需要将采集到的验证码同样分割成单字符，并将字符的值与其二进制文件一一对应映射。

``` php
    /**
     * 处理标本图像
     * @return array
     */
    public function loadTrainData()
    {
        $path = "train/";
        $handler = opendir($path);  
        while (($filename = readdir($handler)) !== false) {  
            if ($filename != "." && $filename != "..") {  
                //将图像名称与图像资源一一对应
                $sampleMap[substr($filename, 0, 1)] = imagecreatefromjpeg($path.$filename);
           }  
        }
        closedir($handler);
        return $sampleMap;
    }
```

### 对比样本
将所要识别的验证码分割出的子单字符图像与样本图像一一对比，像素差距最小的即可能为其值。这种方法在这种简单的验证码中准确率很高。

``` php
    /**
     * 获取单个验证码字符
     * @param  image $img   分割的图像
     * @param  array $sampleMap 标本图像
     * @return string       
     */
    public function getSingleChar($img, $sampleMap)
    {
        $width = imagesx($img);
        $height = imagesy($img);
        $min = $width * $height;
        //将验证码图像与样本图像一一对比，并获取差异最小的样本的值
        foreach ($sampleMap as $filename => $file) {
            $count = 0;
            for ($x=0; $x < $width; $x++) {
                for ($y=0; $y < $height; $y++) {
                    //如果像素点颜色不同，差异值增加
                    if($this->isInterfere($img, $x, $y) != $this->isInterfere($file, $x, $y)) {
                        $count++;
                        if($count >= $min)
                            break 2;
                    }
                }
            }
            if($count < $min) {
                $min = $count;
                $result = $filename;
            }
        }
        return $result;
    }

```

### 总结

这次写简单的验证码识别，接触了很多PHP GD的函数，对于PHP的函数名混乱也有了很深的印象。

其实这个简单的验证码识别中并没有涉及到很多的算法，只是简单的处理图像与分割图像。

还有就是觉得类的设计跟变量名的命名方面还是有待加强。
