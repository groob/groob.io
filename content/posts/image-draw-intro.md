+++
date = "2017-01-07T17:28:58-05:00"
title = "image/draw: Adding Alpha Channel to PNGs."
draft = false

+++

The other day, I was reading a [blog post](http://blog.eriknicolasgomez.com/2016/12/28/managing-sierras-loginwindow-redux/) by Erik Gomez about a recent change Apple made for setting LoginWindow images - as of macOS 10.12, they require a PNG which has an Alpha channel set. This requirement could be annoying and requires tools like Photoshop or ImageMagick.  At a previous job managing lab machines at an art school, I was asked to update both the login window and desktop background image quite often, so I was intrigued by this change. 

I have not had the chance to work with the [image](https://golang.org/pkg/image/) package yet, so I thought this would be a good opportunity to start. The resulting [code](https://github.com/groob/pngalpha) turned out quite elegant, but I did run into a few challenges along the way. 

<!--more-->

# Decoding and Encoding PNG files.

Reading a PNG file in Go is easy. The Decode function expects an io.Reader and returns back an Image, which we can edit.
```
func Decode(r io.Reader) (image.Image, error)
```

Encoding the PNG back to a file is just as simple. Instead of a reader, we specify a writer(usually a file) and the image we want to save. 

```
func Encode(w io.Writer, m image.Image) error
```

I tried opening a PNG file, decoding it and encoding it pack into a new file:

```
	inputFile, err = os.Open("some.png")
	img, err = png.Decode(inputFile)
	outputFile, err = os.Create("new.png")
	err = png.Encode(outputFile, img)
```

But when I did that, I noticed something odd. Although the original already had an alpha channel set, encoding it with `image/png` lost that attribute on the new file.

```
/usr/bin/sips --getProperty hasAlpha some.png
  hasAlpha: yes

/usr/bin/sips --getProperty hasAlpha new.png
  hasAlpha: no
```

Ok, if we can't preserve the original, that's not a good start. Not knowing much about the image package, or the PNG format, I searched around for a bit, and stumbled on a golang-nuts [reply](https://groups.google.com/forum/#!topic/golang-nuts/-hgeEi_7a_g) from Russ Cox to some other png.Encode() question.

{{<figure src="/image-draw/image-draw01.png" title="png.Encode" >}}

So I tried an experiment. I created an RGBA image with the same dimensions as the original, and using `draw.Draw` drew the png image on top of my RGBA image. Then I took the `(0,0)` pixel and changed the value from `255` to `254`. And it worked!

```
	m := image.NewRGBA(src.Bounds())
	draw.Draw(m, m.Bounds(), src, image.ZP, draw.Src)
	t := m.RGBAAt(0, 0)
	t.A = t.A - 1
	m.Set(0, 0, t)
```

I was able to use the above snippet to convert any PNG to one which had an Alpha channel. But something about this code didn't seem right. It felt like a hack. Then, someone on the Gopher Slack team clued me in to a better solution:

> @groob You could make a custom type of `image.RGBA` and implement `Opaque() bool` and always return false.

To understand what this means, we have to look at the [`png.Encode`](https://github.com/golang/go/blob/1ede11d13a2a4ed63e9a6cf8b6039225749fa6ea/src/image/png/writer.go#L510-L515) source. 

{{<figure src="/image-draw/image-draw02.png" title="png.Encode" >}}
When the encoder writes the image as PNG, it first checks to see if the image is `opaque()` and if it is not, it encodes it with a format that has the Alpha channel.

Looking at the definition of `opaque()` we see that it checks if the image satisfties the `opaquer` interface.
```
type opaquer interface {
	Opaque() bool
}
```

{{<figure src="/image-draw/image-draw03.png" title="png.Encode" >}}

In other words, the `opaque()` function will check to see if the Image type has a `Opaque() bool` method and call the method to check if the image is opaque or not.

The `image.RGBA` type has this method, but it scans the entire image pixel by pixel and only returns `false` if the A value is not `255`. We need it to always return `false`. Luckily, in Go we can embed a type inside our own struct and writing our own `Opaque() bool` method.

```
// enforce image.RGBA to always add the alpha channel when encoding PNGs.
type notOpaqueRGBA struct {
	*image.RGBA
}

func (i *notOpaqueRGBA) Opaque() bool {
	return false
}
```

The `notOpaqueRGBA` type we created will call it's own `Opaque()` method, returning `false` every time. Now we can modify the original code to use this new type instead:
```
	img := &notOpaqueRGBA{image.NewRGBA(src.Bounds())}
	draw.Draw(img, img.Bounds(), src, image.ZP, draw.Src)
```

I ended up writing the following helper to convert a file into an Image:

```
func convert(file io.Reader, contentType string) (image.Image, error) {
    // both png.Decode and jpeg.Decode have the same function signature
    // we can decide which to use based on contentType
	var decode func(io.Reader) (image.Image, error)
	switch contentType {
	case "image/jpeg":
		decode = jpeg.Decode
	case "image/png":
		decode = png.Decode
	default:
		return nil, fmt.Errorf("unrecognized image format: %s", contentType)
	}

    // use the decode function to decode the file into a src Image
	src, err := decode(file)
	if err != nil {
		return nil, err
	}

    // declare a `notOpaqueRGBA type with an embeded image.RGBA and
    // draw the source over our image
	img := &notOpaqueRGBA{image.NewRGBA(src.Bounds())}
	draw.Draw(img, img.Bounds(), src, image.ZP, draw.Src)

    // return the image
	return img, nil
}
```


# Detecting the ContentType

You'll notice that `convert` takes a `contentType` param. I used the [`http.DetectContentType`](https://golang.org/pkg/net/http/#DetectContentType) helper from the `net/http` package.

```
// DetectContentType implements the algorithm described
// at http://mimesniff.spec.whatwg.org/ to determine the
// Content-Type of the given data. It considers at most the
// first 512 bytes of data. DetectContentType always returns
// a valid MIME type: if it cannot determine a more specific one, it
// returns "application/octet-stream".
func DetectContentType(data []byte) string 
```

This helper is really neat, and will detect a wide range of MIME types. Since only the first 512 bytes are required, I ended up reading them into a buffer, and resetting the reader back to the starting position.

```
// detectContentType leverages the http.DetectContentType helper. It resets
// the io.ReadSeeker to 0 when done.
func detectContentType(file io.ReadSeeker) (string, error) {
	buf := make([]byte, 512)
	_, err := file.Read(buf)
	if err != nil {
		return "", err
	}

	_, err = file.Seek(0, 0)
	if err != nil {
		return "", err
	}

	return http.DetectContentType(buf), nil
}
```

# Results

I ended up creating a cli utility to convert images: [code](https://github.com/groob/pngalpha), [binary](https://github.com/groob/pngalpha/releases).
I also created a little http service to convert the image directly in your browser: [https://groob.io/mac-login-wp](https://groob.io/mac-login-wp).

This was my first time working with the `image/draw`, `image/png`, and `image/jpeg` packages, and it proved to be a positive experience. I'm continously impressed with how readable the stdandard library code is. Even when I'm working in an unfamiliar domain, Go's type system with implicit interfaces, first class functions and composition makes it simple to reason about my own code and to extend the behavior of external packages.

