# Rails 实现方式学习项目

## 第一部分:
[[原文链接]](https://isotope11.com/blog/build-your-own-web-framework-with-rack-and-ruby-part-1)

### Step 1：新建项目
```zsh
mkdir -p AppName/public/{images, js, css}
touch AppName/{config.ru, public/index.html}
cd AppName && bundle init
```
生成项目结构如下：
```zsh
- AppName
  |- config.ru
  |- Gemfile
  |- public
    |- index.html
    |- images
    |- js
    |- css
```

### Step 2：添加依赖
修改 `Gemfile` 文件后执行 `bundle install` 安装依赖包：
```ruby
# source 'https://rubygems.org'
source 'https://gems.ruby-china.org'   # 因无法访问 https://rubygems.org 而替换为国内镜像
ruby '2.4.0' # 指定 ruby 版本
gem 'rack'   # 引入 rack 包依赖
```
**注意：** 修改 `Gemfile` 需要执行 `bundle install` 重新安装依赖 !

### Step 3：定义处理类
Rack 的使用非常简单。它接受一个对象（ Object ），并在这个对象上调用 `call` 方法，同时传递有关 Web 请求（request）的信息。该方法的返回一个包含响应信息( response )的数组。  
创建 `AppName/lib/request_controller.rb` 并插入如下代码：
```ruby
class RequestController
  def call(env)
    [200, {}, ["Hello World"]]
  end
end
```
分析返回结果：
* 第一个值代表响应代码（response code），此例中为 `200`，其他可用代码诸如： `404 500 301` 等请自行百度。
* 第二个值是一个包含头（header）信息的哈希，此例中为空哈希 `{}`。
* 第三个值是返回信息数组（response data），此例中为 `["Hello World"]`

**注意：** Rails 内部实现与此方式非常相似！因此，记住这几行代码是非常有益的。

### Step 4：使用处理类
我们现在只是创建了这个文件，Rack 并不知道如何处理这个类 `RequestController`，接下来我们需要添加 `AppName/lib/brain_rack.rb` 文件：
```ruby
#!/usr/bin/env ruby
require 'rack'
load 'request_controller.rb' # 引入 request_controller.rb 文件

Rack::Handler::WEBrick.run(
  RequestController.new,     # 实例化 RequestController 类
  Port: 9000                 # 监听 9000 端口
)
```

---
## 第二部分：
[[原文链接]](https://isotope11.com/blog/build-your-own-web-framework-with-rack-and-ruby-part-2)

新建项目结构和修改 `Gemfile` 不再赘述，请参考第一部分。另外，可以在 `Gemfile` 中添加 `gem 'pry'` 或 `gem 'byebug'` 作为断点调试工具（代码中输入 `binding.pry` 或 `byebug` 即可插入断点）。 
### Step 1：创建项目入口
创建 `AppName/config.ru` 并插入如下代码：
```ruby
# 引入 bundler
require 'bundler'

# 使用 Bundler 加载 Gemfile 中添加的所有 gem
Bundler.require

# 引入文件 lib/brain_rack.rb
require File.join(File.dirname(__FILE__),'lib', 'brain_rack')

# 引入文件 lib/request_controller.rb
require File.join(File.dirname(__FILE__),'lib', 'request_controller')

# 引入文件 config/routes.rb （暂未创建）
require File.join(File.dirname(__FILE__),'config', 'routes')

# 实例化 BrainRack 对象
BrainRackApplication = BrainRack.new

# 使用 Rack 运行应用程序
run RequestController.new

```

### Step 2：创建 BrainRack 类
创建 `AppName/lib/brain_rack.rb` 并插入如下代码：
```ruby
require File.join(File.dirname(__FILE__), '..', 'router.rb')

class BrainRack
  attr_reader :router

  def initialize
    @router = Router.new
  end
end
```

### Step 3：创建 RequestController 类
正如第一部分所述，我们需要一个处理请求和返回响应的类 `AppName/lib/request_controller.rb` 插入如下代码：
```ruby
class RequestController
  def call(env)
    route = BrainRackApplication.router.route_for(env)
    if route
      response = route.execute(env)
      return response.rack_response
    else
      return [404, {}, []]
    end
  end
end
```
Rack 将在该请求控制器实例上调用 `call` 方法，并传递一个包含请求信息（路径和方法）的哈希变量 `env`。为了让程序能够响应此请求，我们必须在路由表中配置路由：请求匹配时返回路由，不匹配时则返回“空”（`nil`）。这样，才会在我们的请求匹配路由时，Rack 会得到期望的 `rack_response` 响应。如果没有找到路由（`nil`）则返回一个通用的 `404`。

### Step 4：配置路由
创建 `AppName/config/routes.rb` 插入代码：
```
BrainRackApplication.router.config do
  get "/test", to: "custom#index"  # 仅匹配 "/test" 请求
  get /.*/, to: "custom#show"      # 正则表达式，匹配所有非 "/test" 的请求
end
```

### Step 5：解析路由
创建 `AppName/router.rb` 插入代码：
```ruby
require File.join(File.dirname(__FILE__), 'lib', 'route')

class Router
  attr_reader :routes

  # 使用 Router.new 实例化时，自动调用此方法，初始化一个空哈希 {}
  def initialize
    @routes = Hash.new { |hash, key| hash[key] = [] }
  end

  # 调用 config 方法时，传入的块（block）会自动获得一个隐含的自己
  # get "path", to: "Controller#Action" 实际上是调用自己的 get 方法：
  # router.get("path", to: "Controller#Action")
  def config(&block)
    instance_eval &block
  end

  def get(path, options = {})
    @routes[:get] << [path, parse_to(options[:to])]
  end

  def route_for(env)
    path   = env["PATH_INFO"]
    method = env["REQUEST_METHOD"].downcase.to_sym
    route_array = routes[method].detect do |route|
      case route.first
      when String
        path == route.first
      when Regexp
        path =~ route.first
      end
    end
    return Route.new(route_array) if route_array
    return nil #No route matched
  end

  private
  # 将字符串 "custom#index" 拆分为哈希 { klass: "Custom", method: "index" }  
  # 注意： "Custom" 的首字母被转为大写（详细请百度“类名驼峰命名法”）
  def parse_to(to_string)
    klass, method = to_string.split("#")
    {:klass => klass.capitalize, :method => method}
  end
end
```
下面是 `env` 的代码片段：
```sh
{"SERVER_SOFTWARE"=>"thin 1.6.1 codename Death Proof",
 "SERVER_NAME"=>"localhost",
 "rack.input"=>
  #<Rack::Lint::InputWrapper:0x000000021d0bd0
   @input=#<StringIO:0x0000000354bb88>>,
 "rack.version"=>[1, 0],
 "rack.errors"=>
  #<Rack::Lint::ErrorWrapper:0x000000021d0a90 @error=#<IO:<STDERR>>>,
 "rack.multithread"=>false,
 "rack.multiprocess"=>false,
 "rack.run_once"=>false,
 "REQUEST_METHOD"=>"GET",
 "REQUEST_PATH"=>"/",
 "PATH_INFO"=>"/",
 "REQUEST_URI"=>"/",
 "HTTP_VERSION"=>"HTTP/1.1",
 "HTTP_HOST"=>"localhost:9292",
 "HTTP_CONNECTION"=>"keep-alive",
 "HTTP_ACCEPT"=>
  "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8",
 "HTTP_USER_AGENT"=>
  "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/31.0.1650.57 Safari/537.36",
 "HTTP_ACCEPT_ENCODING"=>"gzip,deflate,sdch",
 "HTTP_ACCEPT_LANGUAGE"=>"en-US,en;q=0.8",
 "GATEWAY_INTERFACE"=>"CGI/1.2",
 "SERVER_PORT"=>"9292",
 "QUERY_STRING"=>"",
 "SERVER_PROTOCOL"=>"HTTP/1.1",
 "rack.url_scheme"=>"http",
 "SCRIPT_NAME"=>"",
 "REMOTE_ADDR"=>"127.0.0.1",
 "async.callback"=>#<Method: Thin::Connection#post_process>,
 "async.close"=>#<EventMachine::DefaultDeferrable:0x000000021c1ba8>}
```

### Step 6：路由映射
创建 `AppName/lib/route.rb` 插入代码：
```ruby
require File.join(File.dirname(__FILE__), '..', 'app', 'controllers', 'base_controller')

class Route
  attr_accessor :klass_name, :path, :instance_method
  def initialize route_array
    @path            = route_array.first
    @klass_name      = route_array.last[:klass]
    @instance_method = route_array.last[:method]
    handle_requires
  end

  def klass
    Module.const_get(klass_name)
  end

  def execute(env)
    klass.new(env).send(instance_method.to_sym)
  end

  def handle_requires
    require File.join(File.dirname(__FILE__), '..', 'app', 'controllers', klass_name.downcase + '.rb')
  end
end
```

### Step 7：基础控制器
创建 `AppName/app/controllers/base_controller.rb` 插入代码：
```ruby
require File.join(File.dirname(__FILE__), '..', '..', 'lib', 'response')

class BaseController
  attr_reader :env

  def initialize env
    @env = env
  end
end
```

### Step 8：自定义控制器
创建 `AppName/app/controllers/custom.rb` 插入代码：
```ruby
class Custom < BaseController
  def index
    Response.new.tap do |response|
      response.body = "Hello World"
      response.status_code = 200
    end
  end

  def show
    Response.new.tap do |response|
      response.body = "Catchall Route"
      response.status_code = 200
    end
  end
end
```

### Step 9：响应方法
创建 `AppName/lib/response.rb` 插入代码：
```ruby
class Response
  attr_accessor :status_code, :headers, :body

  def initialize
    @headers = {}
  end

  def rack_response
    [status_code, headers, Array(body)]
  end
end
```

### Step 10：启动应用
```zsh
rackup config.ru
```
Rack 服务器启动后，在浏览器地址栏输入对应的 IP:port (通常为：localhost:9292)即可访问项目。