---
title: PHP 验证码识别2
date: 2015-11-12 13:30:00
tags:
    - PHP
    - Captcha
---

> 与之前的位置固定的验证码相比，位置不固定的验证码在分割字符这一步步骤难度有所增加。这次增加了验证码的背景颜色亮度判断，使得预处理过后的验证码是黑白二色图，分割字符时根据它们的宽高取剪切。

### 获取验证码的背景颜色亮度
通过统计验证码中各个颜色像素的数量，取最大比例的像素颜色为背景颜色，然后根据阀值，判断颜色的亮度。

在这里可能出现验证码内字符的颜色像素为最大比例，可能会导致后面无法获取各个字符的宽度。这个情况在预处理中加入了处理，

<!-- more -->
``` php
    /**
     * 获取背景颜色亮度
     * @method getBgBright
     * @param  image     $img 验证码图像
     * @return boolean    背景颜色亮度
     */
    public function getBgBright($img)
    {
        $max = 0;
        $bgRgb = null;
        $color = array();

        //获取验证码中背景颜色RGB索引
        for ($x = 0; $x < $this->width; $x++) {
            for ($y = 0; $y < $this->height; $y++) {
                $index = imagecolorat($img, $x, $y);
                @$color[$index]++;
                if($color[$index] > $max){
                    $max = $color[$index];
                    $bgRgb = $index;
                }
            }
        }
        //获取背景RGB颜色
        $bgRgbArray = imagecolorsforindex($img, $bgRgb);
        //与亮度阀值对比
        if ($bgRgbArray['red'] + $bgRgbArray['green'] + $bgRgbArray['blue'] > BRIGHT) {
            return 0;
        }
        return 1;
    }
```

### 预处理图像，转化为黑白二色图
这里添加了两个默认参数，第一个参数`$imgPath`是处理样本验证码的时候所要传的验证码的路径，第二个参数`$bgBright`是当分割字符时无法判断字符宽度的时候，对背景图片亮度取反的操作开关。

``` php
    /**
     * 图像预处理：移除图像背景，转换位黑白二色图
     * @method removeBackgroud
     * @param  string          $imgPath 验证码文件夹路径
     * @return image          黑白二色验证码图像
     */
    public function removeBackgroud($imgPath = null, $bgBright = 1)
    {
        //传入其他图像路径时，处理其他图像
        if(!empty($imgPath)){
            $this->image = $imgPath;
            $this->size = getimagesize($imgPath);
            $this->type = $this->size['mime'];
            $this->width = $this->size[0];
            $this->height = $this->size[1];
        }

        if ($this->type == 'image/png') {
            $img = imagecreatefrompng($this->image);
        } elseif ($this->type == 'image/jpeg') {
            $img = imagecreatefromjpeg($this->image);
        } elseif ($this->type == 'image/gif') {
            $img = imagecreatefromgif($this->image);
        }

        $white = imagecolorallocate($img, 255, 255, 255);
        $black = imagecolorallocate($img, 0, 0, 0);

        //修正背景颜色亮度
        if($bgBright) {
            $this->bgBright = $this->getBgBright($img);
        } else {
            $this->bgBright = !$this->bgBright;
        }

        //对图像中的像素替换为黑白
        for ($x = 0; $x < $this->width; $x++) {
            for ($y = 0; $y < $this->height; $y++) {
                $rgb = imagecolorat($img, $x, $y);
                if ($this->getBright($img, $x, $y) == $this->bgBright) {
                    imagesetpixel($img, $x, $y, $white);
                }else {
                    imagesetpixel($img, $x, $y,$black);
                }
            }
        }
        return $img;
    }
```

### 分割验证码字符
根据预处理后的图像像素分布，确定单个字符的宽高，进行剪切。

``` php
    /**
     * 横向剪切单字符验证码图像
     * @method removeBlankY
     * @param  image       $img   待剪切的验证码图像
     * @param  int       $width 待剪切的验证码图像宽度
     * @return image       剪切后的验证码图像
     */
    public function removeBlankY($img, $width)
    {
        //获取单个验证码字符的字符高度上界
        for ($y = 0; $y < $this->height; $y++) {
            for ($x = 0; $x < $width; $x++) {
                if($this->getBright($img, $x, $y)) {
                    $start = $y;
                    break 2;
                }
            }
        }
        //获取单个验证码字符的字符高度下界
        for ($y = $this->height - 1; $y >= 0; $y--) {
            for ($x = 0; $x < $width; $x++) {
                if($this->getBright($img, $x, $y)) {
                    $end = $y;
                    break 2;
                }
            }
        }

        //根据字符高度横向剪切验证码
        $height = $end - $start +1;
        $subImg = imagecreatetruecolor($width, $height);
        imagecopy($subImg, $img, 0, 0, 0, $start, $width, $height);
        return $subImg;
    }

    /**
     * //获取每列像素中黑色像素的数量
     * @method getWeightlist
     * @param  [type]        $img [description]
     * @return [type]        [description]
     */
    public function getWeightlist($img)
    {
        $weightlist = array();
        for ($x = 0; $x < $this->width; $x++) {
            $count = 0;
            for ($y = 0; $y < $this->height; $y++) {
                if ($this->getBright($img, $x, $y)) {
                    $count++;
                }
            }
            array_push($weightlist, $count);
        }
        return $weightlist;
    }

    /**
     * 分割验证码单个字符
     * @method splitImage
     * @param  image     $img 预处理过的验证码图像
     * @return array     单字符验证码数组
     */
    public function splitImage($img)
    {
        $size = 0;
        $subImgs = array();
        $weightlist = array();

        $weightlist = $this->getWeightlist($img);
        //修正预处理图片
        for ($i = 0; $i < count($weightlist); $i++) {
            if($weightlist[$i] > 0) {
                $size++;
            }
            if($size == count($weightlist)) {
                $img = $this->removeBackgroud(null, 0);
                $weightlist = $this->getWeightlist($img);
            }
        }

        for ($i = 0; $i < count($weightlist);) {
            $length = 0;
            //获取每个字符的宽度
            while($weightlist[$i++] > 1) {
                $length++;
            }

            //根据宽度大小分割验证码
            if ($length > 12) {
                $width = floor($length/2);
                $height = $this->height;

                $subImg1 = imagecreatetruecolor($width, $height);
                imagecopy($subImg1, $img, 0, 0, $i-$length-1, 0, $width, $height);
                $imgs1 = $this->removeBlankY($subImg1, $width);

                $subImg2 = imagecreatetruecolor($width, $height);
                imagecopy($subImg2, $img, 0, 0, $i-$length/2-1, 0, $width, $height);
                $imgs2 = $this->removeBlankY($subImg2, $width);

                array_push($subImgs, $imgs1, $imgs2);
            } elseif ($length > 3) {
                $width = $length;
                $height = $this->height;

                $subImg3 = imagecreatetruecolor($width, $height);
                imagecopy($subImg3, $img, 0, 0, $i-$length-1, 0, $width, $height);
                $imgs = $this->removeBlankY($subImg3, $width);

                array_push($subImgs, $imgs);
            }
        }
        return $subImgs;
    }
```

### 验证码样本获取
对人工识别的验证码进行处理与分割，使图像与其代表的值关联。方便识别的时的样本载入。

``` php
    /**
     * 获取验证码样本数据
     * @method getTrainData
     * @param  string       $imgPath   人工识别的验证码文件路径
     * @param  string       $trainPath 要生成样本验证码数据的文件路径
     * @return null
     */
    public function getTrainData($imgPath, $trainPath)
    {
        $index = 0;
        $handler = opendir($imgPath);
        while (($filename = readdir($handler)) !== false) {
            var_dump($filename);
            if($filename != '.' && $filename != '..') {
                //对人工识别的验证码标本预处理
                $img = $this->removeBackgroud($imgPath.'/'.$filename);
                //对预处理后的验证码进行分割
                $subImgList = $this->splitImage($img);
                //分别对分割后的验证码标本进行保存
                if (count($subImgList) == 4) {
                    for ($i = 0; $i < 4; $i++) {
                        var_dump($filename);
                        $imageName = substr($filename, $i, 1);
                        imagejpeg($subImgList[$i], $trainPath.'/'.$imageName.'-'.++$index.'.jpg');
                        echo $trainPath.'/'.$imageName.'-'.$index.'.jpg<br/>';
                    }
                }
            }
        }
        closedir($handler);
    }

```

### 总结
经过尝试，这个验证码识别类还暂时无法识别沾粘在一起，或者字符扭曲的验证码。

依旧觉得写得跟屎一样。
