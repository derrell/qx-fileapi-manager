# qx-fileapi-manager
Buildingblocks for flexible Filehandling in Qooxdoo

## API Idea

```javascript
var uploadButton = new qx.ui.form.Button('Select File to Upload');
this.add(uploadButton);
var dropZone = new qx.ui.basic.Atom('Drop Your Stuff Here');
this.add(dropZone);
var mgr = new fileapi.Manager(uploadButton,dropZone);
mgr.addListener('dropzoneEnter',function(e){
  var fileList = e.getData().files;
  // https://developer.mozilla.org/en-US/docs/Web/API/FileList
  // visualize
});
mgr.addListener('dropzoneLeave',function(){
  // visualize
});

var makeUploader = function(fileList){
  return new fileapi.Uploader(fileList).set({
    blockSize: 1024**2,
    concurrency: 3,
    retries: 5,
    request: new qx.io.request.XHr("/REST/upload",'POST'),
    uploadCb: function(req,data){ // default upload cb
      // data: { blob: xxx, name: xxx, start: xxx, end: xxx}
      var promise = new qx.Promise(function(resolve,reject){
          req.setRequestData(data);
          req.addListener('success',function(e){
            resolve(e.getData())
          });
          req.addListener('fail',function(e){
            reject(e.getData())
          });
          req.send();
      });
      return promise;
    }
  });
};

mgr.addListener('gotFiles',function(e){
  var fileList = e.getData().files;
  var uploader = makeUploader(fileList);
  uploader.start();
});
```

## Working qooxdoo FileReader

```javascript
// Create a button
var input = new qx.html.Input('file',{
  display: 'none'
},{
  multiple: true,
  accept: 'image/*'
});

var doc = this.getRoot();
/* order matters here ! to make sure the
 * dom node gets created
 */
doc.getContentElement().add(input);


var pick = new qx.ui.form.Button("Upload").set({
  icon: "icon/32/actions/document-open.png"
});

// Add button to document at fixed coordinates
doc.add(pick,
{
  left : 100,
  top  : 50
});

// Add an event listener
pick.addListener("execute", function(e) {
  input.getDomElement().click();
});

var blockSize = 4*1024;
var maxBlocks = 10;
var upload = function (file,block) {
    var reader = new qx.bom.FileReader();
    var end = blockSize;
    if (file.size < blockSize){
      end = file.size;
    }
    reader.addListener('load',function(e){
        this.debug(file.name + " Block "+block ,
          btoa(e.getData().content.substring(0,60)));
        if (end === blockSize && block < maxBlocks){
          upload(file.slice(end,file.size-end),block+1);
        }
    });
    reader.readAsBinaryString(file.slice(0,end));
};

pick.addListenerOnce('appear',function(){
  var el = pick.getContentElement().getDomElement();
  var that = this;
  var target;
  el.addEventListener('dragenter',function(e){
    pick.setIcon("icon/32/actions/go-top.png")
    target = e.target;
    e.stopPropagation();
    e.preventDefault();
  });
  el.addEventListener('dragover',function(e){
    e.stopPropagation();
    e.preventDefault();
  });
  el.addEventListener('dragleave',function(e){
    if (e.target == target){ // ignore child widgets
      e.stopPropagation();
      e.preventDefault();
      pick.setIcon("icon/32/actions/document-open.png");
    }
  });
  el.addEventListener('drop',function(e){
    e.stopPropagation();
    e.preventDefault();
    pick.setIcon("icon/32/actions/document-open.png");
    var dt = e.dataTransfer
    var files = dt.files;
    for (var i=0;i<files.length;i++){
      that.debug(files[i].name,files[i].size,files[i].lastModifiedDate);
      upload(files[i],0);
    }
  },this);
  this.debug('drop zone armed');
},this);



input.addListener('change',function(e){
  var files = input.getDomElement().files;
  for (var i=0;i<files.length;i++){
    upload(files[i],0);
    this.debug(files[i]);
  }
},this);
```
