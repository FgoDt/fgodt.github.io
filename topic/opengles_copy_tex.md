# opengl es 2 纹理拷贝
## 需求
在项目中有时候验证想法或者其他需求时，我们通常会需要对纹理进行拷贝。刚好我最近的项目为了验证问题就详细研究了一下gl里的纹理拷贝
## 调研
### gles v3.2
gles 有很多个版本，其中v3版本自带了一个 [glCopyImageSubData](https://registry.khronos.org/OpenGL-Refpages/es3/html/glCopyImageSubData.xhtml) 从其描述就能看出来该函数的功能 “may be used to copy data from one image (i.e. texture or renderbuffer) to another.”所以在gles v3 使用此函数应该是最优解。
### gles v3.0 v2
但是在很多情况下，我们的应用需要对低版本的gles进行支持，所以大部分情况我们需要一个通用纹理拷贝方法。通用的方式基本就围绕着 [FBO](https://www.khronos.org/opengl/wiki/Framebuffer_Object) 来处理纹理，我们可以通过一下几种方式来处理纹理，得到纹理拷贝效果：
1. 通过 src 纹理创建 FBO，然后通过 [glCopyTexSubImage2D()](https://registry.khronos.org/OpenGL-Refpages/es3.0/html/glCopyTexImage2D.xhtml) 将纹理拷贝到 dst 纹理
2. 通过 dst 纹理创建 FBO，然后将 src 纹理渲染到 FBO
3. 分别使用 src、dst 纹理创建 FBO，然后使用 [glBlitFramebuffer](https://registry.khronos.org/OpenGL-Refpages/es3.0/html/glBlitFramebuffer.xhtml) 将纹理从 src fbo 拷贝到 dst fbo (次方法只能 gles 3.0 以上版本支持）

#### 几种方式性能对比
在[stackoverflow](https://stackoverflow.com/questions/23981016/best-method-to-copy-texture-to-texture)上有人对这几种方式做了对比:

- S=source texture, D=destination texture, SF=FBO of S, DF=FBO of D
- operation=copying texture to texture
- op/s = how many operations for one second(average), larger is better

    Create DF and render S to DF using simple passthrough shader
        945.656op/s (105.747ms for 100 operations)
        947.293op/s (105.564ms for 100 operations)
        949.099op/s (105.363ms for 100 operations)
        949.324op/s (105.338ms for 100 operations)
        948.215op/s (105.461ms for 100 operations)

    Create SF and use glCopyTexSubImage2D() for D
        937.263op/s (106.694ms for 100 operations)
        940.941op/s (106.277ms for 100 operations)
        941.722op/s (106.188ms for 100 operations)
        941.145op/s (106.254ms for 100 operations)
        940.997op/s (106.270ms for 100 operations)

    Create DF and SF and use glBlitFramebuffer()
        828.172op/s (120.748ms for 100 operations)
        843.612op/s (118.538ms for 100 operations)
        845.377op/s (118.290ms for 100 operations)
        847.024op/s (118.060ms for 100 operations)
        843.303op/s (118.581ms for 100 operations)

    Create DF and SF and use glCopyPixels()
        525.711op/s (190.219ms for 100 operations)
        523.396op/s (191.060ms for 100 operations)
        537.605op/s (186.010ms for 100 operations)
        538.560op/s (185.680ms for 100 operations)
        553.059op/s (180.813ms for 100 operations)

从上面数据可以看出时间开销方面：方法1 ～ 方法2 < glBlitFrameBuffer << glCopyPixels
所以从方便的角度考虑的话我们一半可以使用 glCopyTexSubImage2D 方法来拷贝纹理
## glCopyTexSubImage2D 实现
```c
    //创建一个 fbo
    glGenFramebuffers(1, &src_fbo);
   
    glBindFramebuffer(GL_FRAMEBUFFER, src_fbo);
    //将 src 纹理绑定给 fbo
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, src_tex, 0);
   
    //绑定 dst 纹理并拷贝纹理
    glBindTexture(GL_TEXTURE_2D, dst_tex);
    glCopyTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, 0, 0, tex_width, tex_height, 0);
    glBindTexture(GL_TEXTURE_2D, 0);

    //将 src 纹理从 fbo 解绑，已便后续的逻辑操作 src 纹理
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, 0, 0);
    glBindFramebuffer(GL_FRAMEBUFFER, 0);
```
其他方案我并未尝试，因为要么代码量相对多一点，要么性能不是很理想，所以我认为 glCopyTexSubImage2D 基本符合我的需求
