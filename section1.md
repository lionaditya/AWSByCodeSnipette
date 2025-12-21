# 1. AWS IAM 
![alt text](image.png)![alt text](image-1.png)![alt text](image-2.png)![alt text](image-3.png)![alt text](image-4.png)![alt text](image-5.png)![alt text](image-6.png)![alt text](image-7.png)![alt text](image-8.png)![alt text](image-9.png)![alt text](image-10.png)
# 2. AWS S3
![alt text](image-14.png)![alt text](image-15.png)![alt text](image-16.png)![alt text](image-17.png)![alt text](image-18.png)
# 3. AWS S3 with Spring Boot (programmatic Access)
![alt text](image-21.png)![alt text](image-22.png)![alt text](image-23.png)![alt text](image-24.png)![alt text](image-25.png)![alt text](image-26.png)![alt text](image-27.png)![alt text](image-28.png)![alt text](image-29.png)![alt text](image-30.png)![alt text](image-31.png)![alt text](image-32.png)![alt text](image-33.png)![alt text](image-34.png)![alt text](image-35.png)![alt text](image-36.png)![alt text](image-37.png)![alt text](image-38.png)

## pom.xml
```xml
	<dependency>
			<groupId>software.amazon.awssdk</groupId>
			<artifactId>s3</artifactId>
			<version>2.25.14</version>
		</dependency>
```
## S3Config.java
```java
package com.codesnipette.S3DemoApplication.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import software.amazon.awssdk.auth.credentials.AwsBasicCredentials;
import software.amazon.awssdk.auth.credentials.StaticCredentialsProvider;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.s3.S3Client;

@Configuration
public class S3Config {

    @Value("${cloud.aws.region.static}")
    private String region;

    @Value("${cloud.aws.credentials.secret-key}")
    private String secretKey;

    @Value("${cloud.aws.credentials.access-key}")
    private String accessKey;


    /*1)Creating bean for S3Client
        S3-Client should aware which AWS  need to be connects
         There are lots of AWS account - (Require this info: region,secret key and access key)

       2nd) After giving region; We need to provide credentialsProvider
        It takes object of AwsCredentialsProvider
  so for that here we use object of StaticCredentialsProvider.create() method
           where we Pass AWS credentials
     */
    @Bean
    public S3Client s3Client(){

        //3. create object of AWSBasicCredentials
        AwsBasicCredentials awsBasicCredentials = AwsBasicCredentials.create(accessKey,secretKey);

        return S3Client.builder()
                .region(Region.of(region))
                .credentialsProvider(StaticCredentialsProvider.create(awsBasicCredentials))
                .build();
    }
}
```
## app.properties
```properties
spring.application.name=S3DemoApplication


cloud.aws.region.static=ap-south-1
cloud.aws.credentials.secret-key=ToWkD/HsVNVM69jjHmvVRV1tT5ZYOR0KwJaHUn/Y
cloud.aws.credentials.access-key=AKIA4LJ52JZXRZONDK4F

aws.bucket.name=codesnipettev1
```
## S3Service.java
```java
package com.codesnipette.S3DemoApplication.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;
import software.amazon.awssdk.core.ResponseBytes;
import software.amazon.awssdk.core.sync.RequestBody;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.GetObjectRequest;
import software.amazon.awssdk.services.s3.model.GetObjectResponse;
import software.amazon.awssdk.services.s3.model.PutObjectRequest;

import java.io.IOException;

@Service
public class S3Service {

    @Autowired
    private S3Client s3Client; //We use S3Client

    @Value("${aws.bucket.name}")
    private String bucketName;// We use bucketName which is property

    /*
    WE uploadFile() which take MultipartFile
     we execute s3client.putObject()
        putObject take arguments PutObjectRequest
         we just build this PutObjectRequest with bucketName
          and key(what key of your file[orginalFileName] is inside your bucket)

       In 2nd argument we pass file in bytes

       Basically we have file with us that need to be upload in S3
       file size in bytes
       and to upload in S3 we require (bucketName and key)
     */
    public void uploadFile(MultipartFile file) throws IOException {

        s3Client.putObject(PutObjectRequest.builder()
                        .bucket(bucketName)
                        .key(file.getOriginalFilename())
                .build(),
                RequestBody.fromBytes(file.getBytes())
        );
    }

    /*
     Reverse of Upload;
      We have a file in s3 we need to download that file in bytes

      WE require key and bucketName;
     */
    public byte[] downloadFile(String key){
        ResponseBytes<GetObjectResponse> objectAsBytes = s3Client.getObjectAsBytes(GetObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .build());

        return objectAsBytes.asByteArray();
    }

}
```
## S3Controller.java
```java
package com.codesnipette.S3DemoApplication.controller;


import com.codesnipette.S3DemoApplication.service.S3Service;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpHeaders;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;


@RestController
public class S3Controller {

    @Autowired
    private S3Service s3Service;

    /*
    PostRequest Accept file which is Multipart file
     we just upload file via s3Service
     */
    @PostMapping("/upload")
    public ResponseEntity<String> upload(@RequestParam("file") MultipartFile file) throws IOException {
        s3Service.uploadFile(file);
        return ResponseEntity.ok("File Uploaded Successfully");
    }
/*
   Here we download file
   and return the content of file
 */
    @GetMapping("/download/{fileName}")
    public ResponseEntity<byte[]> download(@PathVariable String fileName){
        byte[] data = s3Service.downloadFile(fileName);
        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION,"attachment; fileName="+ fileName)
                .body(data);
    }
}
```
