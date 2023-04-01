This fork add support for Standard Zip Encryption.

The work is based on https://github.com/yeka/zip

Available encryption:

```
zip.StandardEncryption
zip.AES128Encryption
zip.AES192Encryption
zip.AES256Encryption
```

## Warning

Zip Standard Encryption isn't actually secure.
Unless you have to work with it, please use AES encryption instead.

## Buffer Example Encrypt Zip

```go
package main

import (
	"bytes"
	"io"
	"log"
	"os"

	"github.com/TelenLiu/go-zip"
)

func main() {
	contents := []byte("Hello World")
	fzip, err := os.Create(`./test.zip`)
	if err != nil {
		log.Fatalln(err)
	}
	zipw := zip.NewWriter(fzip)
	defer zipw.Close()
	w, err := zipw.Encrypt(`test.txt`, `golang`, zip.AES256Encryption)
	if err != nil {
		log.Fatal(err)
	}
	_, err = io.Copy(w, bytes.NewReader(contents))
	if err != nil {
		log.Fatal(err)
	}
	zipw.Flush()
}
```

## Files Example Encrypt Zip
```go
func ZipFiles(src, dst, pwd string) error {
	fzip, err := os.Create(dst)
	if err != nil {
		log.Fatalln(err)
	}
	zipw := zip.NewWriter(fzip)
	defer zipw.Close()
	//
	rootSrc := src
	return filepath.Walk(rootSrc, func(path string, info os.FileInfo, err error) error {
		if err != nil {
			return err
		}
		if path == rootSrc {
			if info.IsDir() {
				return nil
			}
		}
		
		fpath := strings.Replace(path, rootSrc, "", 1)
		if strings.HasPrefix(fpath, "/") {
			fpath = strings.Replace(fpath, "/", "", 1)
		}

		header, err := zip.FileInfoHeader(info)
		if err != nil {
			return err
		}
		header.Name = strings.TrimPrefix(fpath, filepath.Dir(rootSrc)+"/")
		if info.IsDir() {
			header.Name += "/"
		} else {
			header.Method = zip.Deflate
		}
		// 加密
		header.SetPassword(pwd)
		header.SetEncryptionType(zip.AES256Encryption)
		w, err := zipw.CreateHeader(header)
		if err != nil {
			log.Println(err)
			return err
		}
		if !info.IsDir() {
			file, err := os.Open(path)
			if err != nil {
				return err
			}
			defer file.Close()
			_, err = io.Copy(w, file)
			if err != nil {
				log.Fatal(err)
			}
			zipw.Flush()
		}
		return err
	})
}
```

## Buffer Example Decrypt Zip

```go
package main

import (
	"fmt"
	"io/ioutil"
	"log"

	"github.com/TelenLiu/go-zip"
)

func main() {
	r, err := zip.OpenReader("encrypted.zip")
	if err != nil {
		log.Fatal(err)
	}
	defer r.Close()

	for _, f := range r.File {
		if f.IsEncrypted() {
			f.SetPassword("12345")
		}

		r, err := f.Open()
		if err != nil {
			log.Fatal(err)
		}

		buf, err := ioutil.ReadAll(r)
		if err != nil {
			log.Fatal(err)
		}
		defer r.Close()

		fmt.Printf("Size of %v: %v byte(s)\n", f.Name, len(buf))
	}
}
```

## Files Example Decrypt Zip
```go
func UnZip(zipFile, destDir, pwd string) error {

	zipReader, err := zip.OpenReader(zipFile)
	if err != nil {
		return err
	}
	defer zipReader.Close()

	for _, f := range zipReader.File {
		//加密判断
		if f.IsEncrypted() {
			f.SetPassword(pwd)
			f.SetEncryptionType(zip.AES256Encryption)
		}

		fpath := ""
		name := f.Name
		if find := strings.Contains(name, ":"); find {
			name = name[strings.Index(name, ":")+2 : len(name)]
			fpath = filepath.Join(destDir, name)
		} else {
			fpath = filepath.Join(destDir, name)
		}

		if f.FileInfo().IsDir() {
			os.MkdirAll(fpath, os.ModePerm)
		} else {
			if err = os.MkdirAll(filepath.Dir(fpath), os.ModePerm); err != nil {
				return err
			}

			inFile, err := f.Open()
			if err != nil {
				return err
			}
			defer inFile.Close()

			outFile, err := os.OpenFile(fpath, os.O_WRONLY|os.O_CREATE|os.O_TRUNC, f.Mode())
			if err != nil {
				return err
			}
			defer outFile.Close()

			_, err = io.Copy(outFile, inFile)
			if err != nil {
				return err
			}
		}
	}
	return nil
}
```
