### Requirements
* Users can download all files and these files are packaged into a zip compression file.

### Solutions
1. Add a one-click download button to the [vue]
 file.
2. Add a public route  to the [api.php] file.
3. Retrieve data from SQL, enabling download functionality in the [controller] file.

### Coding

■ Frontend interface

```xxx.vue
     { label: '一括DL', icon: 'fas fa-file-alt', action: function (self) {self.alert(function () {
                location.href = `${mopi.site_dir}/api/public/expenses/buld-download`;
              }, {}, '登録', '一括ダウロードを行います');
            }
          }
```

■ Routes (Public Route)

sources/routes/api.php

```api.php
Route::get('public/xxx/buld-download','Pub\XXXController@bulkDownload');
```

■ Controller

sources/app/Http/Controllers/XxxxController.php

```XxxxController.php
    use ZipArchive;

class XxxxController extends BaseResourceController
{
    public function __construct()
    {
        $this->_setData();
    }

    protected function _setData()
    {
        $this->Model = new Xxxx();
    }
    
    #Retrieve link_uri linked to the desired files from the sysFileIds table.The id field in this table corresponds to the content data in the Xxxxheader table.By extracting the content with specific conditions from Xxxxheader table and can effortlessly obtain the corresponding Uri.
    # Uri———id———content(initial retrieval)
    public function getUri()
    {
        $Xxxxheader = XxxxHeader::where('status', 90)->get();
        $contentValues = $Xxxxheader->pluck('content')->map(function ($content) {
            return explode(',', $content);
        })->flatten()->toArray();
        $sysFileIds = SysFile::whereIn('id', $contentValues)->pluck('id');
        $linkUris = SysFile::whereIn('id', $sysFileIds)->pluck('link_uri');
        return $linkUris->toArray();
    }

    public function bulkDownload()
    {
        $linkUris = $this->getUri();

        #download the temporary files and specify the destination for saving
        $tempDirectory = storage_path('downloads/temp');

            if (!file_exists($tempDirectory)) {
            mkdir($tempDirectory, 0755, true);
        }
        foreach ($linkUris as $linkUri) {
            $filePath = storage_path("app/upload/{$linkUri}");
            if (file_exists($filePath)) {
                $fileName = pathinfo($filePath, PATHINFO_BASENAME);
                $tempFilePath = $tempDirectory . '/' . $fileName;
                copy($filePath, $tempFilePath);
            }
        }

        #package into a zip compression file
        $zipFileName = '証憑.zip';
        $zip = new ZipArchive();
        $zipFilePath = $tempDirectory . '/証憑.zip';
        $zip->open($zipFilePath, ZipArchive::CREATE | ZipArchive::OVERWRITE);
        $files = glob($tempDirectory . '/*');
        foreach ($files as $file) {
            $zip->addFile($file, pathinfo($file, PATHINFO_BASENAME));
        }
        $zip->close();

        #download and save to the users' on a wab page in the browser
        header("Content-Length: " . filesize($zipFilePath));
        header('Content-Disposition: attachment; filename*=UTF-8\'\'' . rawurlencode($zipFileName));
        header("Content-Type: application/octet-stream");
        header("Accept-Ranges: bytes");
        $handle = fopen($zipFilePath,'rb');
        while (!feof($handle)){
            echo fread($handle, 4096);
            if(ob_get_level() > 0) {
                ob_flush();
            }
            flush();
        }
        fclose($handle);

        #delete the temporary files
        $files = glob($tempDirectory . '/*');
        foreach ($files as $file) {
            if (is_file($file)) {
                unlink($file);
            }
        }

        $localFilePath = $zipFilePath;
        return $localFilePath;
    }
}
```