# WordPress
WordPress 是一款风靡全球的 CMS 系统。

<div class="alert alert-info">
    <i class="bi bi-info-circle"></i> 
    WordPress是全球最流行的内容管理系统之一，通过完成这个任务，您将学习如何在平台上部署和配置自己的WordPress博客。
</div>

<h4>准备工作</h4>
<p>在开始部署WordPress之前，您需要了解以下内容：</p>
<ul>
    <li>WordPress需要一个MySQL数据库来存储内容</li>
    <li>两个应用都需要持久化存储来保存数据</li>
    <li>WordPress应用需要开放80端口对外提供服务</li>
</ul>

<h4>部署步骤</h4>
<ol>
    <li><strong>创建存储</strong>
        <ul>
            <li>登录到平台控制台</li>
            <li>进入"存储管理"页面</li>
            <li>创建名为"mysql"的存储，大小至少1GB</li>
            <li>创建名为"wordpress"的存储，大小至少1GB</li>
        </ul>
    </li>
    <li><strong>部署MySQL数据库</strong>
        <ul>
            <li>进入"应用管理"页面</li>
            <li>点击"创建应用"按钮</li>
            <li>选择"有状态应用"</li>
            <li>应用名称填写"mysql"</li>
            <li>容器镜像选择"mysql:8"</li>
            <li>添加环境变量：MYSQL_ROOT_PASSWORD、MYSQL_DATABASE、MYSQL_USER、MYSQL_PASSWORD</li>
            <li>挂载存储：将"mysql"存储挂载到"/var/lib/mysql"路径</li>
            <li>开放端口：3306</li>
            <li>点击"创建"按钮</li>
        </ul>
    </li>
    <li><strong>部署WordPress应用</strong>
        <ul>
            <li>再次点击"创建应用"按钮</li>
            <li>选择"无状态应用"</li>
            <li>应用名称填写"wordpress"</li>
            <li>容器镜像选择"wordpress:latest"</li>
            <li>添加环境变量：WORDPRESS_DB_HOST、WORDPRESS_DB_USER、WORDPRESS_DB_PASSWORD、WORDPRESS_DB_NAME</li>
            <li>挂载存储：将"wordpress"存储挂载到"/var/www/html"路径</li>
            <li>开放端口：80</li>
            <li>点击"创建"按钮</li>
        </ul>
    </li>
    <li><strong>配置域名访问</strong>
        <ul>
            <li>进入"网站管理"页面</li>
            <li>点击"添加网站"按钮</li>
            <li>输入您的域名</li>
            <li>选择后端服务为"wordpress"，端口为80</li>
            <li>点击"确认添加"按钮</li>
        </ul>
    </li>
</ol>

<div class="alert alert-warning">
    <i class="bi bi-exclamation-triangle"></i>
    请注意：首次访问WordPress时，您需要进行初始化设置，包括设置网站标题、管理员用户名和密码等。
</div>

<h4>环境变量参考</h4>
<table class="table table-bordered">
    <thead class="table-light">
        <tr>
            <th>应用</th>
            <th>环境变量</th>
            <th>说明</th>
            <th>示例值</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td rowspan="4"><strong>MySQL</strong></td>
            <td>MYSQL_ROOT_PASSWORD</td>
            <td>MySQL root用户密码</td>
            <td>rootpassword</td>
        </tr>
        <tr>
            <td>MYSQL_DATABASE</td>
            <td>WordPress使用的数据库名</td>
            <td>wordpress</td>
        </tr>
        <tr>
            <td>MYSQL_USER</td>
            <td>WordPress使用的数据库用户</td>
            <td>wpuser</td>
        </tr>
        <tr>
            <td>MYSQL_PASSWORD</td>
            <td>数据库用户密码</td>
            <td>wppassword</td>
        </tr>
        <tr>
            <td rowspan="4"><strong>WordPress</strong></td>
            <td>WORDPRESS_DB_HOST</td>
            <td>MySQL服务地址</td>
            <td>mysql</td>
        </tr>
        <tr>
            <td>WORDPRESS_DB_NAME</td>
            <td>数据库名称</td>
            <td>wordpress</td>
        </tr>
        <tr>
            <td>WORDPRESS_DB_USER</td>
            <td>数据库用户名</td>
            <td>wpuser</td>
        </tr>
        <tr>
            <td>WORDPRESS_DB_PASSWORD</td>
            <td>数据库密码</td>
            <td>wppassword</td>
        </tr>
    </tbody>
</table>