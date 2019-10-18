# OpenGL ES 3.0  Hello_Triangle 

>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

### OpenGL ES 3.0学习汇总
* [OpenGL ES 3.0 学习记录汇总](https://www.jianshu.com/p/216c7e91960a)

---

![Hello_Triangle](http://upload-images.jianshu.io/upload_images/1709375-43328d8d31cab09f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


[官方源代码原始链接](https://github.com/danginsburg/opengles3-book/blob/master/Chapter_2/Hello_Triangle/Hello_Triangle.c)
```
#include "esUtil.h"
typedef struct {
	GLuint programObject;
}TestUserData;

int TestloadShader(GLenum type, const char *shaderSrc) {
	GLuint shader;
	GLuint complied;
	//创建着色器对象
	shader = glCreateShader(type);
	if (shader == 0) {
	    return 0;
	}
	//加载着色器源程序
	glShaderSource(shader, 1, &shaderSrc, NULL);
	//编译着色器源程序
	glCompileShader(shader);

	//检查编译的状态
	glGetShaderiv(shader, GL_COMPILE_STATUS, &complied);

	if (!complied) {
		GLint infoLen = 0;

		glGetShaderiv(shader, GL_INFO_LOG_LENGTH, &infoLen);

		if (infoLen > 1) {
			char *infoLog = malloc(sizeof(char) * infoLen);

			glGetShaderInfoLog(shader, infoLen, NULL, &infoLog);
			esLogMessage("Error compiling shader:\n%s\n", infoLog);

			free(infoLog);
		}
		glDeleteShader(shader);
		return 0;
	}
	return shader;
}

int TestInit(ESContext *esContext) {
	TestUserData *userData = esContext->userData;
	char vShaderStr[] = "#version 300 es \n"
		"layout (location = 0) in vec4 vPosition; \n"
		"void main() { \n"
		"glPosition = vPosition; \n"
		"}";
	char fShaderStr[] = "#version 300 es \n"
		"precision mediump float; \n"
		"out vec4 fragColor; \n"
		"void main() { \n"
		"fragColor = vec4(1.0, 0.0, 0.0, 1.0)"
		"} ";
	GLuint vertexShader;
	GLuint fragmentShader;
	GLuint programObject;
	GLuint linked;

	//加载 顶点 和 片段  着色器
	vertexShader = TestloadShader(GL_VERTEX_SHADER, vShaderStr);
	fragmentShader = TestloadShader(GL_FRAGMENT_SHADER, fShaderStr);

	//创建程序
	programObject = glCreateProgram();

	if (programObject == 0) {
		return 0;
	}


	glAttachShader(programObject, vertexShader);
	glAttachShader(programObject, fragmentShader);

	//链接程序
	glLinkProgram(programObject);
	
	//检查程序链接状态
	glGetProgramiv(programObject, GL_LINK_STATUS, &linked);

	if (!linked) {
		GLint infoLen = 0;

		glGetProgramiv(programObject, GL_INFO_LOG_LENGTH, &infoLen);

		if (infoLen > 1) {
			char *infoLog = malloc(sizeof(char) * infoLen);

			glGetProgramInfoLog(programObject, infoLen, NULL, infoLog);
			esLogMessage("Eroor linking program:\n%s\n", infoLog);

			free(infoLog);
		}
		glDeleteProgram(programObject);
		return FALSE;
	}

	userData->programObject = programObject;

	glClearColor(0.0f, 0.0f, 0.0f, 0.0f);
	return TRUE;
}

void TestDraw(ESContext *esContext) {
	TestUserData *userData = esContext->userData;
	GLfloat vVertices[] = {
		0.0f, 0.5f, 0.0f,
		-0.5f, -0.5f, 0.0f,
		0.5f, 0.5f, 0.0f
	};

	//设置一个窗口
	glViewport(0, 0, esContext->width, esContext->height);

	//清空颜色缓冲区
	glClear(GL_COLOR_BUFFER_BIT);

	//设置为活动程序
	glUseProgram(userData->programObject);

	//加载顶点坐标数据
	glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 0, vVertices);
	glEnableVertexAttribArray(0);

	glDrawArrays(GL_TRIANGLES, 0, 3);

}

void TestShutdown(ESContext *esContext) {
	TestUserData *userData = esContext->userData;
	glDeleteProgram(userData->programObject);
}

int TestesMain(ESContext *esContext) {
	esContext->userData = malloc(sizeof(TestUserData));

	esCreateWindow(esContext, "Hello Triangel", 800, 800, ES_WINDOW_RGB);

	if (!TestInit(esContext)) {
		return GL_FALSE;
	}

	esRegisterShutdownFunc(esContext, TestShutdown);
	esRegisterDrawFunc(esContext, TestDraw);

	return GL_TRUE;
}
```
