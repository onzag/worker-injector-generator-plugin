# worker-injector-generator-plugin
A very simple webpack plugin to generate injection code for workers that share common code with the main web build.

NOTE this is a temporary fix for https://github.com/webpack/webpack/issues/6472 and it's based on the discussion there.

# Usage (Example with hashes)

```javascript
const WorkerInjectorGeneratorPlugin = require("worker-injector-generator-plugin");

module.exports = {
  mode: 'development',
  entry: {
    "service-worker": ["./src/service.worker.ts"],
    "worker": ["./src/worker.ts"],
    "build": ["./src/index.tsx"],
  },
  devtool: 'inline-source-map',
  plugins: [
    new MiniCssExtractPlugin({
      filename: "build.css",
      chunkFilename: "build.css"
    }),
    new webpack.IgnorePlugin(/^\.\/locale$/, /moment$/),
    new WorkerInjectorGeneratorPlugin({
      name: "worker-injector-[hash].js",
      importScripts: [
        "commons-[hash].js",
        "worker-[hash].js",
      ],
    }),
    new WorkerInjectorGeneratorPlugin({
      name: "service-worker-injector-[hash].js",
      importScripts: [
        "commons-[hash].js",
        "service-worker-[hash].js",
      ],
    }),
  ],
  resolve: {
    extensions: ['.ts', '.tsx', '.js', '.mjs']
  },
  optimization: {
    splitChunks: {
      cacheGroups: {
        commons: {
          name: 'commons',
          minChunks: 2,
          chunks: 'initial',
        },
      }
    }
  },
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        exclude: /node_modules/,
        use: {
          loader: "babel-loader"
        },
      },
      {
        test: /\.js$/,
        use: ["source-map-loader"],
        enforce: "pre"
      },
      {
        test: /\.s?css$/,
        use: [
          {
            loader: MiniCssExtractPlugin.loader
          },
          {
            loader: "css-loader"
          },
          {
            loader: "sass-loader"
          }
        ]
      },
      {
        test: /\.(png|jpg|gif|svg|eot|ttf|woff|woff2)$/,
        loader: 'url-loader',
        options: {
          limit: 10000,
        },
      },
      {
        test: /\.mjs$/,
        include: /node_modules/,
        type: 'javascript/auto'
      }
    ]
  },
  output: {
    filename: '[name]-[hash].js',
    path: path.resolve(__dirname, 'dist'),
    libraryTarget: "umd",
    globalObject: "this",
    publicPath: "/rest/resource/",
  }
};
```

You would then include the injector into your build, and add it as a worker (do not add your main worker output as it would hang forever).

The injector then generates the following code (this is for `worker-injector-feb0c7712b38d060b184.js`)

`var base=location.protocol+"//"+location.host+(location.port ? ":"+location.port: "")+"/rest/resource/";window=self;self.importScripts(base+"commons-feb0c7712b38d060b184.js",base+"worker-feb0c7712b38d060b184.js");`

And it will be sent to your output path (in the example above `path.resolve(__dirname, 'dist')`)

Make sure you are using UMD as the library target.

# Options


## options.name (required)

The name of the generated injector worker file, you might include [hash] in the name eg.

```javascript
new WorkerInjectorGeneratorPlugin({
  name: my-injector-[hash].js,
  importScripts: [],
})
```

The file will be generated at your output path.

## options.importScripts (required)

An array of strings to be used as the scripts to be imported, you might include hash in their names as well

```javascript
new WorkerInjectorGeneratorPlugin({
  name: my-injector-[hash].js,
  importScripts: ["javascript-in-common-[hash].js", "my-actual-worker-[hash].js"],
})
```

Notice that the resolution URL in the browser once loaded it's calculated using the output.publicPath so if your public path is eg. `/resources/my-content`

It will resolve to the URL to eg. `http://mysite.com/resources/my-content/javascript-in-common-[hash].js`

## options.publicPath

A public path to override the public path provided by the configuration eg.

```javascript
new WorkerInjectorGeneratorPlugin({
  name: my-injector-[hash].js,
  importScripts: ["javascript-in-common-[hash].js", "my-actual-worker-[hash].js"],
  publicPath: "/workers/",
})
```

Now it will resolve to the URL eg. `http://mysite.com/workers/javascript-in-common-[hash].js`

If no publicPath is set here nor in the output options, the public path will be / that is the root url

`http://mysite.com/javascript-in-common-[hash].js`
