# Laravel application random spike in reponse time

## 问题

持续随机出现极慢的请求(10~20s)，间隔没有明显规律。

## 触发问题的条件

- 使用 file session driver
- session life time 设置的很长

满足以上两个条件后，`storage/framework/sessions` 下的文件将会无限增长，最终磁盘 IO 出现瓶颈，导致极慢的请求。

### gc of file session driver

```
public function gc($lifetime)
{
    $files = Finder::create()
                ->in($this->path)
                ->files()
                ->ignoreDotFiles(true)
                ->date('<= now - '.$lifetime.' seconds');

    foreach ($files as $file) {
        $this->files->delete($file->getRealPath());
    }
}
```

默认情况下每次请求 2%(lottery in config/session.php) 的可能触发 GC