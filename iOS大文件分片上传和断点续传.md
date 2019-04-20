### 总结一下大文件分片上传和断点续传的问题。因为文件过大（比如1G以上），必须要考虑上传过程网络中断的情况。http的网络请求中本身就已经具备了分片上传功能，当传输的文件比较大时，http协议自动会将文件切片（分块），但这不是我们现在说的重点，我们要做的事是保证在网络中断后1G的文件已上传的那部分在下次网络连接时不必再重传。所以我们本地在上传的时候，要将大文件进行分片，比如分成1024*1024B，即将大文件分成1M的片进行上传，服务器在接收后，再将这些片合并成原始文件，这就是分片的基本原理。断点续传要求本地要记录每一片的上传的状态，我通过三个状态进行了标记（wait  loading  finish），当网络中断，再次连接后，从断点处进行上传。服务器通过文件名、总片数判断该文件是否已全部上传完成。
#### 下面来说细节：
> #### 1、首先获取文件（音视频、图片）
#### 分两种情况，一种是在相册库里直接获取，一种是调用相机。如果是通过UIImagePickerView来获取（细节不详述，网上一大堆），我们会发现当你选定一个视频的时候，会出现图1的压缩页面，最后我们的app获取的视频就是这个经过压缩后的视频（不是视频库里的原始视频，这里有个注意点，操作完该压缩视频后记得释放，系统不会帮你释放的，需要你手动来操作，下面会说到），然后通过UIImagePickerView的协议方法中的-
```
(void)imagePickerController:(UIImagePickerController *)picker didFinishPickingMediaWithInfo:(NSDictionary *)info获取视频的Info
fileInfo = {
    UIImagePickerControllerMediaType = "public.movie";
    UIImagePickerControllerMediaURL = "file:///private/var/mobile/Containers/Data/Application/2AAE9E44-0E6D-4499-9AC3-93D44D8342EA/tmp/trim.F36EC46C-4219-43C8-96A7-FA7141AB64D2.MOV";
    UIImagePickerControllerReferenceURL = "assets-library://asset/asset.MOV?id=DEDA9406-3223-4F87-ABB2-98FB5F5EB9C4&ext=MOV";
}
```
#### UIImagePickerControllerMediaType是选取文件的类型，如KUTTypeImage，KUTTypeMovie。这里注意一下movie和video的区别，一个是有声音的视频文件，一个是没有声音的视频文件，当然还有Audio是只有声音没有视频。UIImagePickerControllerMediaURL是视频的URL（如果是相机拍摄的，那么这个就是原始拍摄得到的视频；如果是在相册库里选择的，那就是压缩之后生成的视频），注意这个URL不指向相册库，通过这个URL你可以操作这个视频如删除，拷贝等，可以获取压缩后的视频的大小。UIImagePickerControllerReferenceURL是一个指向相册的URL，官方的解释是an NSURL that references an asset in the AssetsLibrary framework，通过这个URL，你可以获取视频的所有信息，包括文件名，缩略图，时长等（通过ALAssetsLibrary里的assetsLibraryassetForURL:referenceURLresultBlock:）。
#### 如果是相机拍摄的，注意两个保存方法：图片保存到相册
```
assetsLibrarywriteImageDataToSavedPhotosAlbum:UIImageJPEGRepresentation([infovalueForKey:UIImagePickerControllerOriginalImage],(CGFloat)1.0)metadata:nilcompletionBlock: failureBlock:
```
#### 高保真压缩图片的方法
```
NSData * UIImageJPEGRepresentation ( UIImage *image, CGFloat compressionQuality）
```
#### 视频保存到相册：assetsLibrary writeVideoAtPathToSavedPhotosAlbum:MediaURL completionBlock:failureBlock：

#### 到这里，我们就获取了所有需要的文件以及文件信息。下面要做的就是将文件分片。

> ### 2、将获取到的文件分片
#### 首先，我将获取到的文件保存在这这样一个类中
```
@interface CNFile : NSObject
@property (nonatomic,copy)NSString* fileType;//image  or  movie
@property (nonatomic,copy)NSString* filePath;//文件在app中路径
@property (nonatomic,copy)NSString* fileName;//文件名
@property (nonatomic,assign)NSInteger fileSize;//文件大小
@property (nonatomic,assign) NSInteger trunks;//总片数
@property (nonatomic,copy)NSString* fileInfo;
@property (nonatomic,strong)UIImage* fileImage;//文件缩略图
@property (nonatomic,strong) NSMutableArray* fileArr;//标记每片的上传状态
@end
```
#### 这样我们就可以对每一个CNFile对象进行操作了。
```
-(void)readDataWithChunk:(NSInteger)chunk file:(CNFile*)file{
  总片数的获取方法：
    int offset =1024*1024;（每一片的大小是1M）
    NSInteger chunks = (file.fileSize%1024==0)?((int)(file.fileSize/1024*1024)):((int)(file.fileSize/(1024*1024) + 1));
    NSLog(@"chunks = %ld",(long)chunks);
    将文件分片，读取每一片的数据：
    NSData* data;
    NSFileHandle *readHandle = [NSFileHandle fileHandleForReadingAtPath:file.filePath];
    [readHandle seekToFileOffset:offset * chunk];
    data = [readHandle readDataOfLength:offset];
}
```
#### 这样我们就获取了每一片要上传的数据，然后询问服务器，该片是否已经存在
```
-(void)ifHaveData:(NSData*)data WithChunk:(NSInteger)chunk file:(CNFile*)file
```
#### 如果存在，令chunk+1，重复上面的方法读取下一片，直到服务器不存在该片，那么上传该片数据。在这个方法中注意设置该chunk的上传状态（wait  loading finish），这将关系到本地判断该文件是否已全部上传完成。

#### 下一步就是上传的过程：
```
-(void)uploadData:(NSData*) data WithChunk:(NSInteger) chunk file:(CNFile*)file；
```
#### 在服务器返回该片上传成功后，我们要做的事有很多：
* #### 1）先将已经成功上传的本片的flag置finish
```
[file.fileArr replaceObjectAtIndex:chunk withObject:@“finish"];
```

* #### 2）查看是否所有片的flag都已经置finish，如果都已经finishi，说明该文件上传完成，那么删除该文件，上传下一个文件或者结束。
```
for (NSInteger j =0; j<chunks; j++){
if (j == chunks || ((j == chunks -1)&&([file.fileArr[j]isEqualToString:@"finish"])))
     [me deleteFile:file.filePath];
     [me readNextFile];
}
```
* #### 3）如果没有都finish，那么看本地下一chunk对用的flag是否是wait
```
 NSLog(@"查看第%ld片的状态",chunk+1);
 for(NSInteger i = chunk+1;i < chunks;i++)
  {
     NSString* flag = [file.fileArrobjectAtIndex:i];
      if ([flagisEqualToString:@"wait"]) {
             [me readDataWithChunk:ifileName:fileNamefile:file];
               break;
          }
   }
```
#### 在第2、3步之间可以有一个 2.5）判断是否暂停上传
```
if(me.isPause ==YES)
  {
  //将目前读到了第几个文件的第几片保存到本地
     [self saveProgressWithChunk:chunk file:file];
      return ;
   }
```
#### 这个操作实际上和上传过程中断网是一样的，为了断点续传，在断网或者暂停的时候，我们要将目前的进度保存起来，以便下次上传时略过前面已置finish的片。
#### 然后还有一个问题，如果我们就这样线性的一片一片上传，实际上失去了分片上传的意义，应该结合多线程，使分片上传过程并发执行，同时上传多片，这样就提高了上传效率，并充分利用了网络带宽。
```
    dispatch_async(dispatch_queue_t queue, ^{
        [me readDataWithChunk: chunk];
    })
```
#### 最后注意一下，每上传完一个视频，去设置里看看你的app占用的存储空间有没有增大哦，如果你没有处理那个生成的压缩视频，你会发现你的app的空间占用量是很大的。

#### 转自：[iOS大文件分片上传和断点续传](http://blog.csdn.net/nndasdfg/article/details/51436731)