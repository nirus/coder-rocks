
Above is the youtube video link ID that can be embedded in the `claim.json`. For the legend image of the article page it's
hardcoded as **hero.jpg**. Upload a legend image with this filename and its automatically picked up by the compiler.

This is an example article to demonstrate all possible markdown styles and custom mods available.

### Inline code
This is how the inline code looks like `01/28/2020`. 

### Screenshot or figure display
Now an example for screenshot with a caption on the article.

![{ caption: "[ This is a sample screen shot ]", width: "500px" }](./screenshot.png)

Format like `{ caption: "[ This is a sample screen shot ]", width: "500px" }` is sent as an attribute to the `<img />` inside the `<figure />` tag. `{ caption: "My custom caption"}` is placed inside the `<figcaption />`

### Code

Below is the example javascript code block with syntax highlighting.

```js
const todayWithoutTime = new Date(
  new Date().getFullYear(),
  new Date().getMonth(),
  new Date().getDate()
);
console.log("todayWithoutTime: ", todayWithoutTime);
// todayWithoutTime:  Tue Jan 28 2020 00:00:00 GMT-0500 (Eastern Standard Time)
```
### Quote display
Below is an example of a quoted text with credits linking back to author article.

> A well-designed system makes it easy to do the right things and annoying (but not impossible) to do the wrong things
> <div class="quote-credit"><p>– Jeff Atwood</p><a target="_blank" href="https://blog.codinghorror.com/falling-into-the-pit-of-success/">Falling Into The Pit of Success</a></div>
