# 优化代码结构，多物体运动
```html
<div class="page-header"><h3>9、优化代码结构，多物体运动</h3></div>

<canvas id = "test09-canvas" width = "800" height = "600"></canvas>

<br/>
<input type="checkbox" id="twinkle" /> 发亮
<br/>
(逗号/句号 控制 靠近/远离，W/S 控制旋转)

<script id = "shader-vs" type = "x-shader/x-vertex">
	attribute vec3 aVertexPosition;
	attribute vec2 aTextureCoord;

	uniform mat4 uMVMatrix;
	uniform mat4 uPMatrix;
	varying vec2 vTextureCoord;

	void main(void)
	{
		gl_Position = uPMatrix * uMVMatrix * vec4(aVertexPosition, 1.0);
		vTextureCoord = aTextureCoord;
	}
</script>

<script id = "shader-fs" type = "x-shader/x-fragment">
	precision mediump float;
	varying vec2 vTextureCoord;

	uniform sampler2D uSampler;

	uniform vec3 uColor;

	void main(void)
	{
		vec4 textureColor = 
			texture2D(uSampler, vec2(vTextureCoord.s, vTextureCoord.t));
		gl_FragColor = 
			textureColor * vec4(uColor, 1.0);
	}
</script>
<script type="text/javascript">
	requestAnimFrame = window.requestAnimationFrame || 
		window.mozRequestAnimationFrame || 
		window.webkitRequestAnimationFrame || 
		window.msRequestAnimationFrame || 
		window.oRequestAnimationFrame || 
		function(callback) { setTimeout(callback, 1000 / 60); }; 
</script>
<script type="text/javascript">

$(document).ready(function ()
{
	webGLStart();
});

function webGLStart()
{
	var canvas = $("#test09-canvas")[0];
	initGL(canvas);
	initShaders();
	initBuffers();
	initTexture();

	initWorldObjects();

	gl.clearColor(0.0, 0.0, 0.0, 1.0);

	$(document).keydown(handleKeyDown);
	$(document).keyup(handleKeyUp);
}
function tick()
{
	requestAnimFrame(tick);
	handleKeys();
	drawScene();
	animate();
}

var gl;
function initGL(canvas)
{
	try
	{
		gl = canvas.getContext("webgl") || canvas.getContext("experimental-webgl");
		gl.viewportWidth = canvas.width;
		gl.viewportHeight = canvas.height;
	}catch(e){}
	if(!gl)
	{
		alert("无法初始化“WebGL”。");
	}
}

var starTexture;
function initTexture()
{
	starTexture = gl.createTexture();
	starTexture.image = new Image();
	starTexture.image.onload = function()
	{
		handleLoadedTexture(starTexture);
		tick();
	}
	starTexture.image.src = "/Public/image/star.gif";
}
function handleLoadedTexture(texture)
{
	gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, 1);
	

	gl.bindTexture(gl.TEXTURE_2D, texture);
	gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, 
		gl.UNSIGNED_BYTE, texture.image);
	gl.texParameteri(gl.TEXTURE_2D, 
		gl.TEXTURE_MAG_FILTER, gl.LINEAR);
	gl.texParameteri(gl.TEXTURE_2D, 
		gl.TEXTURE_MIN_FILTER, gl.LINEAR);
	
	gl.bindTexture(gl.TEXTURE_2D, null);
}
function getShader(gl, id)
{
	var shaderScript = $("#" + id);
	if(!shaderScript.length)
	{
		return null;
	}
	var str = shaderScript.text();

	var shader;
	if(shaderScript[0].type == "x-shader/x-fragment")
	{
		shader = gl.createShader(gl.FRAGMENT_SHADER);
	}
	else if(shaderScript[0].type == "x-shader/x-vertex")
	{
		shader = gl.createShader(gl.VERTEX_SHADER);
	}
	else
	{
		return null;
	}
	gl.shaderSource(shader, str);
	gl.compileShader(shader);
	if(!gl.getShaderParameter(shader, gl.COMPILE_STATUS))
	{
		alert(gl.getShaderInfoLog(shader));
		return null;
	}
	return shader;
}

var shaderProgram;
function initShaders()
{
	var fragmentShader = getShader(gl, "shader-fs");
	var vertexShader = getShader(gl, "shader-vs");
	
	shaderProgram = gl.createProgram();
	gl.attachShader(shaderProgram, vertexShader);
	gl.attachShader(shaderProgram, fragmentShader);
	gl.linkProgram(shaderProgram);

	if(!gl.getProgramParameter(shaderProgram, gl.LINK_STATUS))
	{
		alert("无法初始化“Shader”。");
	}
	gl.useProgram(shaderProgram);

	shaderProgram.vertexPositionAttribute = 
		gl.getAttribLocation(shaderProgram, "aVertexPosition");
	gl.enableVertexAttribArray(shaderProgram.vertexPositionAttribute);

	shaderProgram.textureCoordAttribute = 
		gl.getAttribLocation(shaderProgram, "aTextureCoord");
	gl.enableVertexAttribArray(shaderProgram.textureCoordAttribute);

	shaderProgram.pMatrixUniform = 
		gl.getUniformLocation(shaderProgram, "uPMatrix");
	shaderProgram.mvMatrixUniform = 
		gl.getUniformLocation(shaderProgram, "uMVMatrix");
	shaderProgram.samplerUniform = 
		gl.getUniformLocation(shaderProgram, "uSampler");
	shaderProgram.colorUniform = 
		gl.getUniformLocation(shaderProgram, "uColor");

}


var mvMatrix = mat4.create();

var mvMatrixStack = [];

var pMatrix = mat4.create();

function mvPushMatrix()
{
	var copy = mat4.clone(mvMatrix);
	mvMatrixStack.push(copy);
}
function mvPopMatrix()
{
	if(mvMatrixStack.length == 0)
	{
		throw "不合法的矩阵出栈操作!";
	}
	mvMatrix = mvMatrixStack.pop();
}

function setMatrixUniforms()
{
	gl.uniformMatrix4fv(shaderProgram.pMatrixUniform, false, pMatrix);
	gl.uniformMatrix4fv(shaderProgram.mvMatrixUniform, false, mvMatrix);
}


var starVertexPositionBuffer;
var starVertexTextureCoordBuffer;

function initBuffers()
{
	starVertexPositionBuffer = gl.createBuffer();
	gl.bindBuffer(gl.ARRAY_BUFFER, starVertexPositionBuffer);
	vertices = [
		-1.0, -1.0,  0.0,
		 1.0, -1.0,  0.0,
		-1.0,  1.0,  0.0,
		 1.0,  1.0,  0.0
		];
	gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(vertices), 
		gl.STATIC_DRAW);
	starVertexPositionBuffer.itemSize = 3;
	starVertexPositionBuffer.numItems = 4;

	starVertexTextureCoordBuffer = gl.createBuffer();
	gl.bindBuffer(gl.ARRAY_BUFFER, starVertexTextureCoordBuffer);
	var textureCoords = [
		0.0, 0.0,
		1.0, 0.0,
		0.0, 1.0,
		1.0, 1.0
		];
	gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(textureCoords), 
		gl.STATIC_DRAW);
	starVertexTextureCoordBuffer.itemSize = 2;
	starVertexTextureCoordBuffer.numItems = 4;
}


var currentlyPressedKeys = {};
function handleKeyDown(event)
{
	currentlyPressedKeys[event.keyCode] = true;
}
function handleKeyUp(event)
{
	currentlyPressedKeys[event.keyCode] = false;
}

var zoom  = -15;
var tilt = 90;
var spin = 90;
function handleKeys()
{
	if(currentlyPressedKeys[190])
	{
		//"."/">"句号键
		zoom -= 0.1;
	}
	if(currentlyPressedKeys[188])
	{
		//","/"<"逗号键
		zoom += 0.1;
	}
	if(currentlyPressedKeys[87])
	{
		//W
		tilt += 2;
	}
	if(currentlyPressedKeys[83])
	{
		//S
		tilt -= 2;
	}	
}
function drawScene()
{
	gl.viewport(0, 0, gl.viewportWidth, gl.viewportHeight);
	gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);

	mat4.perspective(pMatrix, 45, 
		gl.viewportWidth / gl.viewportHeight, 0.1, 100.0);

	gl.blendFunc(gl.SRC_ALPHA, gl.ONE);
	gl.enable(gl.BLEND);

	mat4.identity(mvMatrix);

	mat4.translate(mvMatrix, mvMatrix, [0.0, 0.0, zoom]);

	mat4.rotate(mvMatrix, mvMatrix, degToRad(tilt), [1, 0, 0]);
	
	var twinkle = $("#twinkle").is(":checked");
	for(var i in stars)
	{
		stars[i].draw(tilt, spin, twinkle);
		spin += 0.1;
	}
}
var stars = [];
function initWorldObjects()
{
	var numStars = 50;
	for(var i = 0; i < numStars; i ++)
	{
		stars.push(new Star((i / numStars) * 5.0, i / numStars));
	}
}
//==========
//Star类
function Star(startingDistance, rotationSpeed)
{
	this.angle = 0;
	this.dist = startingDistance;
	this.rotationSpeed = rotationSpeed;
	//设置一个起始颜色
	this.randomiseColors();
}
Star.prototype.draw = function(tilt, spin, twinkle)
{
	mvPushMatrix();
	//移动到位
	mat4.rotate(mvMatrix, mvMatrix, degToRad(this.angle), [0.0, 1.0, 0.0]);
	mat4.translate(mvMatrix, mvMatrix, [this.dist, 0.0, 0.0]);
	//旋转
	mat4.rotate(mvMatrix, mvMatrix, degToRad(-this.angle), [0.0, 1.0, 0.0]);
	mat4.rotate(mvMatrix, mvMatrix, degToRad(-tilt), [1.0, 0.0, 0.0]);
	if(twinkle)
	{
		//画一个不旋转的star
		gl.uniform3f(shaderProgram.colorUniform, 
			this.twinkleR, this.twinkleG, this.twinkleB);
		drawStar();
	}
	//所有Star围绕Z旋转
	mat4.rotate(mvMatrix, mvMatrix, degToRad(spin), [0.0, 0.0, 1.0]);
	//画颜色
	gl.uniform3f(shaderProgram.colorUniform, this.r, this.g, this.b);
	drawStar();
	mvPopMatrix();
};

var effectiveFPMS = 60 / 1000;
Star.prototype.animate = function(elapsedTime)
{
	this.angle += this.rotationSpeed * effectiveFPMS * elapsedTime;
	//逐步减小与中心的距离
	this.dist -= 0.01 * effectiveFPMS * elapsedTime;
	if(this.dist < 0.0)
	{
		this.dist += 5.0;
		this.randomiseColors();
	}
};

Star.prototype.randomiseColors = function()
{
	//给星星一个随机的颜色
	this.r = Math.random();
	this.g = Math.random();
	this.b = Math.random();
	//如果打开了发亮开关，则多绘制一次，用另一个随机的颜色作为亮光
	this.twinkleR = Math.random();
	this.twinkleG = Math.random();
	this.twinkleB = Math.random();
};
//==========

function drawStar()
{
	gl.activeTexture(gl.TEXTURE0);
	gl.bindTexture(gl.TEXTURE_2D, starTexture);
	gl.uniform1i(shaderProgram.samplerUniform, 0);

	gl.bindBuffer(gl.ARRAY_BUFFER, starVertexTextureCoordBuffer);
	gl.vertexAttribPointer(shaderProgram.textureCoordAttribute, 
		starVertexTextureCoordBuffer.itemSize, gl.FLOAT, false, 0, 0);

	gl.bindBuffer(gl.ARRAY_BUFFER, starVertexPositionBuffer);
	gl.vertexAttribPointer(shaderProgram.vertexPositionAttribute, 
		starVertexPositionBuffer.itemSize, gl.FLOAT, false, 0, 0);

	setMatrixUniforms();
	gl.drawArrays(gl.TRIANGLE_STRIP, 0, starVertexPositionBuffer.numItems);
}

function degToRad(degrees)
{
	return degrees * Math.PI / 180;
}
var lastTime = 0;
function animate()
{
	var timeNow = new Date().getTime();
	if(lastTime != 0)
	{
		var elapsed = timeNow - lastTime;

		for(var i in stars)
		{
			stars[i].animate(elapsed);
		}
	}
	lastTime = timeNow;
}
</script>
```