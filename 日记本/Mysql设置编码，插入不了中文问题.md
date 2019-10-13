问题出现在 安装的时候没有自己设置编码，用的默认的latin1编码

windows
修改步骤：
* 找到安装的根目录,找到与bin文件夹同级的my.ini文件
* 打开文件 修改 default-character-set=latin1为 default-character-set=utf8
* 往下翻 修改 character-set-server=latin1 为 character-set-server=utf8
* 然后 win + R  ，输入services.msc 找到mysql服务 重新启动

这样登录数据库的时候用命令status去查的时候发现编码都是utf-8了，
然后运行use test(数据库名字) ，再运行status,发现Db的字符集还是latin1
然后修改方法是运行下面的命令
* 修改库的编码 alter database test（数据库名字） character set utf8;

然而用jdbc插入中文的时候 同样报错，心累!

然后 发现eclipse的编码是GBK,
然后修改为UTF-8之后，运行,还是抛出异常
com.mysql.jdbc.exceptions.MySQLSyntaxErrorException: Unknown column '0å“ˆå“ˆ' in 'field list'
心累....

解决了在记录吧...

最后附上代码：

java文件

    package com.luoxiaoke.learn.jdbc;

    import java.sql.Connection;
    import java.sql.DriverManager;
    import java.sql.ResultSet;
    import java.sql.Statement;
    import java.util.ResourceBundle;

    public class JDBCWithConfig {
    	public static void main(String[] args) throws Exception {
		String classname;
		String url;
		String user;
		String password;
		ResourceBundle bundle = ResourceBundle.getBundle("DBconfig");
		classname = bundle.getString("driverclass");
		url = bundle.getString("url");
		user = bundle.getString("user");
		password = bundle.getString("password");

		Class.forName(classname);
		Connection con = DriverManager.getConnection(url, user, password);
		Statement statement = con.createStatement();

		for (int i = 0; i < 10; i++) {
			statement.execute("insert into user values(null," + i+"哈哈," + i + ")");
		}

		String sql = "select * from user";
		ResultSet resultSet = statement.executeQuery(sql);
		while (resultSet.next()) {
			System.out.println(resultSet.getString("name") + "___"+ resultSet.getInt("age"));
		}

		resultSet.close();
		statement.close();
		con.close();
  	}
    }


DBconfig.properties

    driverclass = com.mysql.jdbc.Driver
    url = jdbc:mysql:///lxk
    user = root
    password = root

