## What is Storybook ?
[Storybook](https://storybook.js.org/) is an open source tool for developing UI components in isolation for React, Vue, and Angular. It makes building stunning UIs organized and efficient.

### Now let’s move straight to the custom webpack config
[Link](https://storybook.js.org/docs/configurations/custom-webpack-config/#docs-content) from the storybook documentation explains the custom configuration for the webpack. But wait! it doesn’t work if you do as explained in the documentation (Bummer!). I have logged an issue explaining this.

Issue has two parts to it. One explaining about their default documentation and another one that partially works using **MiniCssExtractPlugin**. Visit the issue [here](https://github.com/storybookjs/storybook/issues/11052) to know more, detailed info on what i am talking about.

<h2 align="center">Now back to the topic</h2>

**Goal use case** – To implement a **common SCSS constants** to be used across the stories (colors.scss, utils.scss) without repetitive imports inside individual stories.

So besides the issue i spoke earlier, i had to do a work around to get the project moving with the above goal.

### Final woking solution (hours of effort!)

Finally i solved it by using [sass-resources-loader](https://www.npmjs.com/package/sass-resources-loader) to load the common SCSS constants by mutating config object. Below is my working webpack configuration for my storybook.

```js
webpackFinal: async (config, { configType }) => {
    // `configType` has a value of 'DEVELOPMENT' or 'PRODUCTION'
    // You can change the configuration based on that.
    // 'PRODUCTION' is used when building the static version of storybook.

    config.module.rules.map((rule) => {
      if (rule.oneOf) {
        rule.oneOf = rule.oneOf.slice().map((subRule) => {
          if (subRule.test instanceof RegExp && subRule.test.test('.scss')) {
            return {
              ...subRule,
              use: [
                ...subRule.use,
                {
                  loader: require.resolve('sass-resources-loader'),
                  options: {
                    resources: [
                      path.resolve(__dirname, '../src/styles/_common.scss')
                    ]
                  }
                }
              ],
            }
          }
          return subRule;
        });
      }
      return rule;
    });
    return config;
  },
```

Full working project with above config can be found on [GitHub](https://github.com/nirus/storybook-issue/tree/work-arround) & this [link](https://github.com/nirus/storybook-issue/blob/597a4df091362dbd61422d7033ef210bdd146f20/.storybook/main.js#L13) for the webpack configuration.

### What exactly did we do?
- Run a map (loop) on the incoming config object inside webpackFinal method.
- Check for default [oneOf](https://webpack.js.org/configuration/module/#ruleoneof) rule set for ***.scss*** file.
- Inject our file of interest ie., ***_common.scss***.
- Voila we have successfully injected our config!

## Feedback
> Let me know your thoughts and any solutions that you encountered. I will be happy to include with this article.