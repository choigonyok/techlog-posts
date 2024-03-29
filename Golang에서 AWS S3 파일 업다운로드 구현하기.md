[ID: 57]
[Tags: PROJECTS GOLANG DEV CLOUD]
[Title: Golang에서 AWS S3 파일 업다운로드 구현하기]
[WriteTime: 2024-01-18]
[ImageNames: 81326447-1234-45be-bd98-5e6bf3707d15.png]
[Subtitle: 블로그의 이미지 파일들을 AWS S3로 관리하자!]

		
## Content

1. Preamble
2. Golang 구현

## 1. Preamble


기존 테크 블로그 아키텍처에서 백엔드 서버는 하나의 단일 인스턴스에서 실행되었다. 테크 블로그 게시글에 사용되는썸네일 등의 이미지 파일들은 인스턴스 내부 디렉토리에 저장해서 관리하고 있었다.

이 방식은 두 가지 단점이 존재했다.


1. 서버에 장애가 생겨 재시작해야할 때, 인스턴스 내부의 이미지 파일이 모두 사라진다.
2. Multi-AZ 아키텍처에서는 같은 이미지 파일 셋을 모든 인스턴스가 가지고있어야하기 때문에, 이미지 파일을 디렉토리에서 관리하는 것은 비효율적이다.

EKS로 클러스터를 이전하고, 고가용성을 보장하기 위해 Multi-AZ 아키텍처를 도입함에 따라 새로운 이미지 파일 관리 방식이 필요했고, AWS의 S3 버킷을 사용하기로 했다.

## 2. Golang 구현


```go
package image

import (
	"fmt"
	"mime/multipart"
	"os"

	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/credentials"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/s3"
	"github.com/aws/aws-sdk-go/service/s3/s3manager"
)

const (
	region       = "ap-northeast-2"
	s3BucketName = "blog-choigonyok"
)

var (
	awsSecretKey = os.Getenv("AWS_SECRET_KEY")
	awsAccessKey = os.Getenv("AWS_ACCESS_KEY")
)
```


image package를 생성하고, 변수와 상수를 정의해서 이후에 S3 버킷명이나, 리전, AWS 크리덴셜이 변경되어도 수정이 복잡해지지 않도록 구현했다.

### Upload


```go
func Upload(f *multipart.FileHeader, imageName string) error {
	sess, err := session.NewSession(&aws.Config{
		Region: aws.String(region),
		Credentials: credentials.NewStaticCredentials(
			awsAccessKey,
			awsSecretKey,
			""),
	})
	if err != nil {
		return err
	}
	uploader := s3manager.NewUploader(sess)

	file, _ := f.Open()
	if err != nil {
		fmt.Println("Error copying the file")
		return err
	}

	file.Seek(0, 0)

	result, err := uploader.Upload(&s3manager.UploadInput{
		Bucket: aws.String(s3BucketName),
		Key:    aws.String(imageName),
		Body:   file,
	})
	if err != nil {
		return fmt.Errorf("failed to upload file, %v", err)
	}
	fmt.Printf("file uploaded to, %s\\\\\\\\n", aws.StringValue(&result.Location))
	return nil
}
```


S3에 이미지 파일을 저장할 때, 키값으로 이미지 이름을 지정했다. 이미지 이름은 게시글이 등록될 때 `랜덤스트링.{확장자}`로 이미지를 저장하게된다.

원본 파일명을 그대로 사용하면, 여러 게시글에서 같은 이미지 파일이 사용될 떄, 게시글 하나를 삭제하면 S3 버킷에서 해당 이미지 파일이 삭제되어서 다른 게시글에서도 이미지가 삭제될 것을 방지하고자 랜덤스트링을 생성해 이미지 파일명으로 사용하도록 구현했다.

`*multipart.FileHeader` 타입의 f를 파라미터로 전달해서 `Open()`메서드로 파일을 열고,  `Seek()` 메서드를 사용해서 파일 디스크립터를 맨 앞으로 이동시켜준다.

그럼 `Upload()` 메서드를 통해서 해당 파일이 읽어지면서 S3 버킷에 파일이 저장된다.

```go
...
formData, err := c.MultipartForm()
if err != nil {
    resp.Response500(c, err)
    return
}

postDatas := formData.Value["data"]

<이미지 파일 외 게시글 데이터 처리>

imageDatas := formData.File["file"]

for i, v := range imageDatas {
    image.ImageName = data.CreateRandomString() + v.Filename[strings.LastIndex(v.Filename, "."):]
    if i == 0 {
        image.Thumbnail = "true"
    } else {
        image.Thumbnail = "false"
    }

    err = img.Upload(v, image.ImageName)
    if err != nil {
        fmt.Println(err.Error())
    }
  
    <DB에 이미지 데이터 저장>
  
    imageNames = append(imageNames, image.ImageName)
}
...
```


클라이언트에서 multipart/form-data 형태로 전송한 요청을 gin의 `c.MultipartForm()` 메서드로 파싱했다. `data` 키로는 본문, 제목, 태그 등의 텍스트 데이터를 파싱하고, `file` 키로는 이미지 파일 데이터를 파싱했다.

이미지들을 돌면서 이미지의 이름을 `랜덤스트링 + 원본 이미지의 확장자` 로 수정해주고, 첫 번째 등록된 이미지는 썸네일로 DB에 저장하기 위해 `image.Thumbnail` 필드를 true로 설정해주었다.

`c.MultipartForm()` 메서드는 `*multipart.FileHeader` 타입 슬라이스를 반환하기 때문에, 슬라이스 내부 엘리먼트를 range문으로 돌면서 Upload 함수를 호출해 S3 버킷에 이미지를 하나씩 저장했다.

### Download


다운로드는 S3 버킷에 있는 파일을 Key를 통해 읽어오는 것을 의미한다. 이미지 파일을 읽어서 브라우저에 출력하기 위한 목적으로 작성되었다.

```go
func Download(imageName string) (*s3.GetObjectOutput, error) {
	sess, err := session.NewSession(&aws.Config{
		Region: aws.String(region),
		Credentials: credentials.NewStaticCredentials(
			awsAccessKey,
			awsSecretKey,
			""),
	})
	if err != nil {
		return nil, err
	}

	svc := s3.New(sess)

	output, err := svc.GetObject(&s3.GetObjectInput{
		Bucket: aws.String(s3BucketName),
		Key:    aws.String(imageName),
	})
	if err != nil {
		fmt.Println("ERROR:  ", err.Error())
	}

	if err != nil {
		return nil, fmt.Errorf("failed to download file, %v", err)
	}
	fmt.Printf("file downloaded")
	return output, nil
}
```


다운로드는 S3 버킷에 있는 파일을 Key를 통해 읽어오는 걸 의미한다.

`GetObject()` 메서드로 S3 버킷 이름과 키를 지정해서 `*s3.GetObjectOutput` 타입의 output을 리턴받는다.

이 output은 아래와 같이 여러 필요한 핸들러에서 `io.Writer` 인터페이스 타입인 gin의 `c.Writer` 에 `io.Copy()` 되어서 클라이언트에게 응답된다.

```go
...
image, err := img.Download(thumbnailName)
if err != nil {
    resp.Response500(c, err)
    return
}
defer image.Body.Close()

c.Header("Content-Type", *image.ContentType)
io.Copy(c.Writer, image.Body)
if err != nil {
    resp.Response500(c, err)
    return
}
...
```


### Remove


Remove는 게시글이 삭제될 때, 해당 게시글에서 사용되었던 더 이상 사용되지 않을 이미지 파일들의 데이터도 함께 삭제하는 목적으로 작성되었다.

```go
func Remove(imageName string) error {
	sess, err := session.NewSession(&aws.Config{
		Region: aws.String(region),
		Credentials: credentials.NewStaticCredentials(
			awsAccessKey,
			awsSecretKey,
			""),
	})
	if err != nil {
		return err
	}

	svc := s3.New(sess)

	_, err = svc.DeleteObject(&s3.DeleteObjectInput{
		Bucket: aws.String(s3BucketName),
		Key:    aws.String(imageName),
	})
	if err != nil {
		return err
	}
	return nil
}

```


`DeleteObject()` 메서드로 S3 버킷 이름과, DB에서 읽어온 이미지 파일 명을 Key로 사용해서 S3 버킷에 있는 이미지 파일을 삭제한다.

```go
...
imageNames, err := svcMaster.DeletePostByPostID(postID)
if err != nil {
    resp.Response500(c, err)
    return
}

for _, v := range imageNames {
    if err := img.Remove(v); err != nil {
        resp.Response500(c, err)
        return
    }
}
...
```


`DeletePostByPostID` 메서드는 게시글 데이터를 DB에서 삭제하고, 게시글에서 사용된 모든 이미지들의 이름을 문자열 슬라이스 형태로 리턴한다.

`imageNames`를 돌면서 Remove 함수를 호출해서 하나씩 삭제해주는 방식으로 구현했다.
