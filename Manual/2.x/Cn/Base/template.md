## 模板引擎集成

框架秉承高度自由的理念，既可以作为API框架，也可以作为常规的全站框架，开发混合式Web服务，本例介绍了如何集成当下常用的三种模板引擎，为框架集成View层，提供渲染模板视图的能力

|      引擎名称      |           说明           |                   仓库地址                   |                  开发参考手册                  |
| :------------: | :--------------------: | :--------------------------------------: | :--------------------------------------: |
|     Smarty     |    业界最著名的PHP模板引擎之一     | [GitHub](https://github.com/smarty-php/smarty) | [官方文档](https://www.smarty.net/docs/zh_CN/) |
| think-template | ThinkPHP 5.1 官方分离的模板引擎 | [GitHub](https://github.com/top-think/think-template) | [官方手册](https://www.kancloud.cn/manual/thinkphp5/118122) |
|     Blade      |     Laravel的模板引擎组件     | [GitHub](https://github.com/duncan3dc/blade) | [官方手册](https://laravel-china.org/docs/laravel/5.6/blade) |

## 引入支持库

无论使用哪个模板引擎，首先都要引入模板引擎的支持库文件，我们推荐使用 Composer 进行第三方组件包的管理，请确保已经正确安装了 Composer 然后执行下面对应的命令，引入引擎的支持库包

```bash
# 使用Smarty引擎
composer require smarty/smarty

# 使用think-template
composer require topthink/think-template

# 使用Blade
composer require duncan3dc/blade
```

> 第三方模板引擎支持库不属于框架的维护范围，如果遇到问题，请首先考虑升级到最新版本，如果问题无法解决，请到群里求助



## 建立自定义控制器抽象类

为了简化渲染页面的操作，我们需要建立一个自定义的控制器抽象类，以便自动初始化模板引擎，以及对输出方法进行封装，简化操作。下面给出的三段代码例子，分别针对不同的模板引擎，请按照自己使用的引擎参照对应的代码建立抽象控制器，请注意模板引擎都需要编译缓存目录以及模板视图目录，请提前新建好对应的目录

下面所示的例子中，视图控制器抽象类都直接存放在 APP 文件夹中，如果放在其他文件夹中，请注意命名空间，确保能自动加载到文件

### Smarty

```php
<?php

namespace App;

use EasySwoole\Config;
use EasySwoole\Core\Http\AbstractInterface\Controller;
use EasySwoole\Core\Http\Request;
use EasySwoole\Core\Http\Response;

/**
 * 视图控制器
 * Class ViewController
 * @author  : evalor <master@evalor.cn>
 * @package App
 */
abstract class ViewController extends Controller
{
    protected $view;

    /**
     * 初始化模板引擎
     * ViewController constructor.
     * @param string   $actionName
     * @param Request  $request
     * @param Response $response
     */
    function __construct(string $actionName, Request $request, Response $response)
    {
        $this->view = new \Smarty();
        $tempPath   = Config::getInstance()->getConf('TEMP_DIR');    # 临时文件目录
        $this->view->setCompileDir("{$tempPath}/templates_c/");      # 模板编译目录
        $this->view->setCacheDir("{$tempPath}/cache/");              # 模板缓存目录
        $this->view->setTemplateDir(EASYSWOOLE_ROOT . '/Views/');    # 模板文件目录
        $this->view->setCaching(false);
      
        parent::__construct($actionName, $request, $response);
    }

    /**
     * 输出模板到页面
     * @param  string|null $template 模板文件
     * @author : evalor <master@evalor.cn>
     * @throws \Exception
     * @throws \SmartyException
     */
    function fetch($template = null)
    {
        $content = $this->view->fetch($template);
        $this->response()->write($content);
        $this->view->clearAllAssign();
        $this->view->clearAllCache();
    }

    /**
     * 添加模板变量
     * @param array|string $tpl_var 变量名
     * @param mixed        $value   变量值
     * @param boolean      $nocache 不缓存变量
     * @author : evalor <master@evalor.cn>
     */
    function assign($tpl_var, $value = null, $nocache = false)
    {
        $this->view->assign($tpl_var, $value, $nocache);
    }
}
```

### Think-Template

```php
<?php

namespace App;

use EasySwoole\Config;
use EasySwoole\Core\Http\AbstractInterface\Controller;
use EasySwoole\Core\Http\Request;
use EasySwoole\Core\Http\Response;
use think\Template;

/**
 * 视图控制器
 * Class ViewController
 * @author  : evalor <master@evalor.cn>
 * @package App
 */
abstract class ViewController extends Controller
{
    protected $view;

    /**
     * 初始化模板引擎
     * ViewController constructor.
     * @param string   $actionName
     * @param Request  $request
     * @param Response $response
     */
    function __construct(string $actionName, Request $request, Response $response)
    {
        $this->view = new Template();
        $tempPath   = Config::getInstance()->getConf('TEMP_DIR');     # 临时文件目录
        $this->view->config([
            'view_path'  => EASYSWOOLE_ROOT . '/Views/',              # 模板文件目录
            'cache_path' => "{$tempPath}/templates_c/",               # 模板编译目录
        ]);
      
        parent::__construct($actionName, $request, $response);
    }

    /**
     * 输出模板到页面
     * @param  string|null $template 模板文件
     * @param array        $vars     模板变量值
     * @param array        $config   额外的渲染配置
     * @author : evalor <master@evalor.cn>
     */
    function fetch($template, $vars = [], $config = [])
    {
        ob_start();
        $this->view->fetch($template, $vars, $config);
        $content = ob_get_clean();
        $this->response()->write($content);
    }
}
```

### Blade

```php
<?php

namespace App;

use EasySwoole\Config;
use EasySwoole\Core\Http\AbstractInterface\Controller;
use EasySwoole\Core\Http\Request;
use EasySwoole\Core\Http\Response;
use duncan3dc\Laravel\BladeInstance;

/**
 * 视图控制器
 * Class ViewController
 * @author  : evalor <master@evalor.cn>
 * @package App
 */
abstract class ViewController extends Controller
{
    protected $view;

    /**
     * 初始化模板引擎
     * ViewController constructor.
     * @param string   $actionName
     * @param Request  $request
     * @param Response $response
     */
    function __construct(string $actionName, Request $request, Response $response)
    {
        $tempPath   = Config::getInstance()->getConf('TEMP_DIR');    # 临时文件目录
        $this->view = new BladeInstance(EASYSWOOLE_ROOT . '/Views', "{$tempPath}/templates_c");
        
        parent::__construct($actionName, $request, $response);
    }

    /**
     * 输出模板到页面
     * @param string $view
     * @param array  $params
     * @author : evalor <master@evalor.cn>
     */
    function render(string $view, array $params = [])
    {
        $content = $this->view->render($view, $params);
        $this->response()->write($content);
    }
}
```

### 进行模板渲染

建立一个测试控制器进行模板渲染，请提前建立好模板文件，这里假设你已经有了一个index模板文件

```php
<?php

namespace App\HttpController;

use App\ViewController;

/**
 * Class Index
 * @author  : evalor <master@evalor.cn>
 * @package App\HttpController
 */
class Index extends ViewController
{
    function index()
    {
        // Blade View
        $this->render('index');     # 对应模板: Views/index.blade.php

        // Think-Template
        $this->fetch('index');      # 对应模板: Views/index.html

        // Smarty
        $this->fetch('index.html'); # 对应模板: Views/index.html
    }
}
```



## 静态资源文件处理

由于 URL Dispatcher 的解析，直接使用 Swoole 作为HTTP服务器，在默认情况下并不能处理静态文件的返回，需要编辑全局配置文件，加入额外的配置项如下：

```php
<?php

return [
    'MAIN_SERVER' => [
        'HOST'        => '0.0.0.0',
        'PORT'        => 9501,
        'SERVER_TYPE' => \EasySwoole\Core\Swoole\ServerManager::TYPE_WEB_SERVER,
        'SOCK_TYPE'   => SWOOLE_TCP,
        'RUN_MODEL'   => SWOOLE_PROCESS,
        'SETTING'     => [
            'worker_num'            => 8,     // worker进程
            'max_request'           => 5000,  // 执行5000次后重启worker进程
            'task_worker_num'       => 8,     // 异步任务进程
            'task_max_request'      => 10,    // 执行10次后重启task进程
            // 加入以下两条配置以返回静态文件
            'document_root'         => EASYSWOOLE_ROOT.'/Public',  // 静态资源目录
            'enable_static_handler' => true,
        ],
    ]

    // 以下配置省略.......
];
```

> 注意静态资源目录需要提前新建好 改为自己的目录 **！！！不要直接复制！！！**
> 不建议使用document_root,在swoole 4.2.13之前的版本都有着重大的bug,访问者可通过127.0.0.1:9501/../../../../../usr/local/php/etc/php.ini去访问服务器所有的文件


## 前端服务器

由于 Swoole Server 对 HTTP 协议的支持并不完整，建议仅将 EasySwoole 作为后端服务，并且在前端增加 NGINX 或 APACHE 作为代理，参照下面的例子添加转发规则

### NGINX

```nginx
server {
    root /data/wwwroot/;
    server_name local.swoole.com;
    location / {
        proxy_http_version 1.1;
        proxy_set_header Connection "keep-alive";
        proxy_set_header X-Real-IP $remote_addr;
        if (!-e $request_filename) {
             proxy_pass http://127.0.0.1:9501;
        }
    }
}
```

### APACHE

```nginx
<IfModule mod_rewrite.c>
  Options +FollowSymlinks
  RewriteEngine On
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteCond %{REQUEST_FILENAME} !-f
  #RewriteRule ^(.*)$ index.php/$1 [QSA,PT,L]  fcgi下无效
  RewriteRule ^(.*)$  http://127.0.0.1:9501/$1 [QSA,P,L]
   #请开启 proxy_mod proxy_http_mod request_mod
</IfModule>
```

