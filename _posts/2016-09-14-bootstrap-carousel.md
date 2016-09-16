---
layout:     post
title:      Bootstrap Carousel Display
date:       2016-09-14 12:55:19
summary:    modify bootstrap carousel to display previous and next images at once
categories: developing
---

When you’re trying to build your own website and using Bootstrap carousel, you may be tempted to do a little different, like this one:

<img src="/images/bootstrap-carousel.png" />

It displays the previous and next images simultaneously and let me tell how to do this step by step.

## Dependencies

* Bootstrap 3.3.7

## Create a basic carousel

First, we add a default carousel to our website.

```html
<div id="myCarousel" class="carousel slide" data-ride="carousel">
  <!-- Wrapper for slides -->
  <div class="carousel-inner" role="listbox">
    <div class="item active">
      <img src="img_wheat.jpg" alt="Wheat">
    </div>
    <div class="item">
      <img src="img_farm.jpg" alt="Farm">
    </div>
    <div class="item">
      <img src="img_grain.jpg" alt="Grain">
    </div>
    <div class="item">
      <img src="img_harvest.jpg" alt="Harvest">
    </div>
  </div>

  <!-- Left and right controls -->
  <a class="left carousel-control" href="#myCarousel" role="button" data-slide="prev">
    <span class="glyphicon glyphicon-chevron-left" aria-hidden="true"></span>
    <span class="sr-only">Previous</span>
  </a>
  <a class="right carousel-control" href="#myCarousel" role="button" data-slide="next">
    <span class="glyphicon glyphicon-chevron-right" aria-hidden="true"></span>
    <span class="sr-only">Next</span>
  </a>
</div>
```

<br>
Add Javascript to trigger the carousel start.

```javascript
<script>
  $('.carousel').carousel({
    interval: 5000 //changes the speed
  })
</script>
```


It’s pretty easy and you can find a tutorial [here](http://www.w3schools.com/bootstrap/bootstrap_carousel.asp).

## Display multiple images

Now we modify our carousel to display 3 images at once.

We change carousel item into responsiveness.

```html
<div id="myCarousel" class="carousel slide" data-ride="carousel">
  <!-- Wrapper for slides -->
  <div class="carousel-inner" role="listbox">
    <div class="item active">
      <div class="col-xs-4">
        <img src="img_wheat.jpg" class="img-responsive">
      </div>
    </div>
    <div class="item">
      <div class="col-xs-4">
        <img src="img_farm.jpg" class="img-responsive">
      </div>
    </div>
    <div class="item">
      <div class="col-xs-4">
        <img src="img_grain.jpg" class="img-responsive">
      </div>
    </div>
    <div class="item">
      <div class="col-xs-4">
        <img src="img_harvest.jpg" class="img-responsive">
      </div>
    </div>
  </div>

  <!-- Left and right controls -->
  <a class="left carousel-control" href="#myCarousel" role="button" data-slide="prev">
    <span class="glyphicon glyphicon-chevron-left" aria-hidden="true"></span>
    <span class="sr-only">Previous</span>
  </a>
  <a class="right carousel-control" href="#myCarousel" role="button" data-slide="next">
    <span class="glyphicon glyphicon-chevron-right" aria-hidden="true"></span>
    <span class="sr-only">Next</span>
  </a>
</div>
```

<br>
Then we use JQuery to clone the following images.

```javascript
<script>
  $('.carousel .item').each(function(){
    var next = $(this).next();
    if (!next.length) {
      next = $(this).siblings(':first');
    }
    next.children(':first-child').clone().appendTo($(this));
                                             
    if (next.next().length>0) {
      next.next().children(':first-child').clone().appendTo($(this));
    } else {
      $(this).siblings(':first').children(':first-child').clone().appendTo($(this));
    }
  });
</script>
```

The default bootstrap will display one image at once, this image will become the first one. Then we clone the next image as second one. If the next image doesn’t exist, it means the first image is the last image. If so, we clone the first image as second one. It’s the same concept on the third image, we just change our base image from the first image to the second image.

After that, it will look like this:
<img src="/images/bootstrap-carousel_2.png" />

It looks great, but it still slides three images at once. If we just want to slide one image, how to do that?

We add some CSS to our website.

```css
/* for chrome */
@media screen and (-webkit-min-device-pixel-ratio:0)
{
  .carousel-inner > .item.next,
  .carousel-inner > .item.active.right {
    -webkit-transform: translate3d(33%, 0, 0);
    transform: translate3d(33%, 0, 0);
  }
  .carousel-inner > .item.prev,
  .carousel-inner > .item.active.left {
    -webkit-transform: translate3d(-33%, 0, 0);
    transform: translate3d(-33%, 0, 0);
  }
}
/* for firefox */
@-moz-document url-prefix() {
  .carousel-inner .next,
  .carousel-inner .active.right {
    left: 33%;
  }
  .carousel-inner .prev,
  .carousel-inner .active.left {
    left: -33%;
  }
}
```

It means we change the previous and next image positions and makes our carousel look like slide one image at once.

## Display one image with preview

Eventually, we only display one image with the previous and next fragments.

Basically, we expand carousel to be larger than browser and hide the overflow.

```css
.carousel {
  overflow: hidden;
  }
.carousel-inner {
  width: 200%;
  left: -50%;
}
```

## Reference

[How to display previous and next images with a Bootstrap carousel](http://stackoverflow.com/a/31570790)
