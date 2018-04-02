Title: WebGL Skinning
Description: How to skin a mesh in WebGL

Given WebGL is basically just a programming language (GLSL) with inputs there's a million ways to do skinning.

The basic part is that you need bones (which is just a fancy way of saying a matrix hierarchy) and weights. Weights are per vertex values that go from 0 to 1 to say how much a particular bone-matrix affects the position of that vertex. Weights are kind of like vertex colors as far as data. One set of weights per vertex. In other words the weights are put in a buffer and provided through attributes.

Typically you limit the number of weights per vertex partly because otherwise it would be way too much data. A character can have anywhere from 15 bones (Virtual Fighter 1) to 150-300 bones (some modern games). If you had 300 bones you'd need 300 weights PER vertex PER bone. So instead most realtime skinning systems limit it ~4 weights per vertex. Usually this is accomplished in an exporter/converter that takes data from a 3D packages like blender/maya/3dsmax and for each vertex finds the 4 bones with the highest weights and then normalizing those weights.

To give an pseudo example a non-skinned vertex is typically computed like this

    gl_Position = projection * view * model * position;

A skinned vertex is effectively computed like this

    gl_Position = projection * view *
                  (bone1Matrix * position * weight1 +
                   bone2Matrix * position * weight2 +
                   bone3Matrix * position * weight3 +
                   bone4Matrix * position * weight4);

Assuming you stored the bones matrices in a uniform array and the weights and which bone each weight applies to you might do something like

    varying vec4 weights;           // 4 weights per vertex
    varying vec4 boneNdx;           // 4 bone indices per vertex
    uniform mat4 bones[MAX_BONES];  // 1 matrix per bone

    gl_Position = projection * view *
                  (bones[int(boneNdx[0])] * position * weight[0] +
                   bones[int(boneNdx[1])] * position * weight[1] +
                   bones[int(boneNdx[2])] * position * weight[2] +
                   boneS[int(boneNdx[3])] * position * weight[3]);

Of course there's a limit to the number of uniforms you can use. The lower limit on WebGL is 64 vec4s which is only 8 mat4s and you probably need some of those uniforms for other things.  A couple ways around that. One is to pre-process the models offline and break them into multiple parts each one using no more than N bones. Another is to store the bone matrices in a texture.

The one other big issue is what's called the *Bind Pose*. Here's the issue. Let's say you have a model of a person with the origin (0,0,0) on the floor just between their feet.

      ___O___           head and head 'bone' are 2.0 units above origin
         |               ^
         |               |
        / \              |
    ___/_._\____       __|__
         ^
         +--- origin

Now imagine you put a matrix/bone/join at their head and you want to use that for bone for skinning. To keep it simple imagine you just set the weights so the the vertices of the head have a 1.0 weight for the head bone and no other joints influence those vertices. There's a problem. The head vertices are 2 units above the origin. The head bone is also 2 units above the origin. If you actually multiplied those head vertices by the head bone matrix you'd get vertices 4 units above the origin. The original 2 units of the vertices + the 2 units of the head bone matrix.

There are various solutions. A common one is to store a "bind pose" which is an extra matrix per joint of where each matrix before you used it to influence the vertices. In that case the bind pose of the head matrix would be 2 units above the origin. So now you can use the inverse of that matrix to subtract out the extra 2 units.

In other words the bone matrices passed to the shader have each been multiplied by their inverse bind pose so as to make their influence only how much they changed from their original positions relative to the origin of the mesh.

Let's make a small example. We'll animate in 2d a model like this

    0-----2-----4-----6-----8
    |     |     |     |     |
    b0    |     b1    |     b2
    |     |     |     |     |
    1-----3-----5-----7-----9

* Where `b0`, `b1`, and `b2` are the bone matrices.
* `b1` is a child of `b0` and `b2` is a child of `b1`
* Verts `0,1` will get a weight of 1.0 from bone b0
* Verts `2,3` will get a weight of 0.5 from bones b0 and b1
* Verts `4,5` will get a weight of 1.0 from bone b1
* Verts `6,7` will get a weight of 0.5 from bones b1 and b2
* Verts `8,9` will get a weight of 1.0 from bone b3


<!-- begin snippet: js hide: false console: false babel: false -->

<!-- language: lang-js -->

    "use strict";
    var m4 = twgl.m4;
    var gl = document.getElementById("c").getContext("webgl");

    // compiles and links the shaders, looks up attribute and uniform locations
    var programInfo = twgl.createProgramInfo(gl, ["vs", "fs"]);

    var arrays = {
      position: {
        size: 2,
        data: [
         0,  1,  // 0
         0, -1,  // 1
         2,  1,  // 2
         2, -1,  // 3
         4,  1,  // 4
         4, -1,  // 5
         6,  1,  // 6
         6, -1,  // 7
         8,  1,  // 8
         8, -1,  // 9
        ],
      },
      boneNdx: {
        size: 4,
        data: [
          0, 0, 0, 0,  // 0
          0, 0, 0, 0,  // 1
          0, 1, 0, 0,  // 2
          0, 1, 0, 0,  // 3
          1, 0, 0, 0,  // 4
          1, 0, 0, 0,  // 5
          1, 2, 0, 0,  // 6
          1, 2, 0, 0,  // 7
          2, 0, 0, 0,  // 8
          2, 0, 0, 0,  // 9
        ],
      },
      weight: {
        size: 4,
        data: [
         1, 0, 0, 0,  // 0
         1, 0, 0, 0,  // 1
        .5,.5, 0, 0,  // 2
        .5,.5, 0, 0,  // 3
         1, 0, 0, 0,  // 4
         1, 0, 0, 0,  // 5
        .5,.5, 0, 0,  // 6
        .5,.5, 0, 0,  // 7
         1, 0, 0, 0,  // 8
         1, 0, 0, 0,  // 9
        ],
      },

      indices: [
        0, 1,
        0, 2,
        1, 3, //
        2, 3,
        2, 4,
        3, 5, //
        4, 5,
        4, 6,
        5, 7, //
        6, 7,
        6, 8,
        7, 9,
        8, 9,
      ]
    };
    // calls gl.createBuffer, gl.bindBuffer, gl.bufferData
    var bufferInfo = twgl.createBufferInfoFromArrays(gl, arrays);

    var boneArray = new Float32Array(4 * 16);

    var uniforms = {
      projection: m4.ortho(-20, 20, -10, 10, -1, 1),
      view: m4.translation([-6, 0, 0]),
      bones: boneArray,
      color: [1, 0, 0, 1],
    };

    // make views for each bone. This lets all the bones
    // exist in 1 array for uploading but as separate
    // arrays for using with the math functions
    var boneUniforms = [];  // the uniform data
    var bones = [];         // the value before multiplying by inverse bind matrix
    var bindPose = [];      // the bind matrix
    for (var i = 0; i < 4; ++i) {
      boneUniforms.push(new Float32Array(boneArray.buffer, i * 4 * 16, 16));
      bindPose.push(m4.identity());  // just allocate storage
      bones.push(m4.identity());     // just allocate storage
    }

     // rotate each bone by a and simulate a hierarchy
     function computeBoneMatrices(bones, a) {
      var m = m4.identity();
      m4.rotateZ(m, a, bones[0]);
      m4.translate(bones[0], [4, 0, 0], m);
      m4.rotateZ(m, a, bones[1]);
      m4.translate(bones[1], [4, 0, 0], m);
      m4.rotateZ(m, a, bones[2]);
      // bones[3] is not used
    }

    // compute the initial positions of each matrix
    computeBoneMatrices(bindPose, 0);

    // compute their inverses
    var bindPoseInv = bindPose.map(function(m) {
      return m4.inverse(m);
    });

    function render(time) {
      twgl.resizeCanvasToDisplaySize(gl.canvas);
      gl.viewport(0, 0, gl.canvas.width, gl.canvas.height);

      var t = time * 0.001;
      var a = Math.sin(t) * 0.8;
      computeBoneMatrices(bones, a);

      // multiply each by its bindPoseInverse
      bones.forEach(function(bone, ndx) {
        m4.multiply(bone, bindPoseInv[ndx], boneUniforms[ndx]);
      });

      gl.useProgram(programInfo.program);
      // calls gl.bindBuffer, gl.enableVertexAttribArray, gl.vertexAttribPointer
      twgl.setBuffersAndAttributes(gl, programInfo, bufferInfo);

      // calls gl.uniformXXX, gl.activeTexture, gl.bindTexture
      twgl.setUniforms(programInfo, uniforms);

      // calls gl.drawArrays or gl.drawIndices
      twgl.drawBufferInfo(gl, bufferInfo, gl.LINES);

      drawAxis(uniforms.projection, uniforms.view, bones);

      requestAnimationFrame(render);
    }
    requestAnimationFrame(render);


    // --- ignore below this line - it's not relevant to the answer and it's a bad example ---

    var axisProgramInfo;
    var axisBufferInfo;
    function drawAxis(projection, view, bones) {
      if (!axisProgramInfo) {
        axisProgramInfo = twgl.createProgramInfo(gl, ['vs2', 'fs']);
        axisBufferInfo  = twgl.createBufferInfoFromArrays(gl, {
          position: {
            size: 2,
            data: [
              0, 0,
              1, 0,
            ],
          },
        });
      }

      var uniforms = {
        projection: projection,
        view: view,
      };

      gl.useProgram(axisProgramInfo.program);
      twgl.setBuffersAndAttributes(gl, axisProgramInfo, axisBufferInfo);

      for (var i = 0; i < 3; ++i) {
        drawLine(bones[i], 0, [0 ,1, 0, 1]);
        drawLine(bones[i], Math.PI * 0.5, [0, 0, 1, 1]);
      }

      function drawLine(mat, angle, color) {
        uniforms.model = m4.rotateZ(mat, angle);
        uniforms.color = color;
        twgl.setUniforms(axisProgramInfo, uniforms);
        twgl.drawBufferInfo(gl, axisBufferInfo, gl.LINES);
      }
    }

<!-- language: lang-css -->

    canvas { border: 1px solid black; }

<!-- language: lang-html -->

    <script id="vs" type="notjs">
    attribute vec4 position;
    attribute vec4 weight;
    attribute vec4 boneNdx;

    uniform mat4 projection;
    uniform mat4 view;
    uniform mat4 bones[4];

    void main() {

      gl_Position = projection * view *
                    (bones[int(boneNdx[0])] * position * weight[0] +
                     bones[int(boneNdx[1])] * position * weight[1] +
                     bones[int(boneNdx[2])] * position * weight[2] +
                     bones[int(boneNdx[3])] * position * weight[3]);

    }
    </script>
    <script id="fs" type="notjs">
    precision mediump float;
    uniform vec4 color;
    void main () {
      gl_FragColor = color;
    }
    </script>
    <script id="vs2" type="notjs">
    attribute vec4 position;

    uniform mat4 projection;
    uniform mat4 view;
    uniform mat4 model;

    void main() {
      gl_Position = projection * view * model * position;
    }
    </script>

    <script src="https://twgljs.org/dist/2.x/twgl-full.min.js"></script>
    <canvas id="c"><canvas>

<!-- end snippet -->

The red lines are the *skinned* mesh. The green and blue lines represent the x-axis and y-axis of each bone or "joint".

NOTE: bones vs joints is confusing. There's only 1 thing, *matrices*. But, in a 3d modelling package they usually draw a gizmo (a ui widget) between each matrix. That looks ends up looking like a bone. The joints are where matrices are.

One other thing to note, the example above is using floats for the weights and the bone indices but you could easily use `UNSIGNED_BYTE` to save a bunch of space.

Also note that one way to look at this stuff in action is to use the [Shader Editor extension for Chrome](https://chrome.google.com/webstore/detail/shader-editor/ggeaidddejpbakgafapihjbgdlbbbpob?hl=en) or [Firefox](https://developer.mozilla.org/en-US/docs/Tools/Shader_Editor)

Looking at [this three.js example](http://threejs.org/examples/webgl_animation_skinning_blending.html) I can see the bone code in the shader.

[![using shader editor to view bone techniques][1]][1]

Here I can see the code is either using uniforms or a texture for the bone matrices



  [1]: http://i.stack.imgur.com/9PUNq.jpg
