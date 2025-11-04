---
date: 2025-05-21
tags:
  - Spring
  - jdbc
sticker: lucide//alert-triangle
---
>Repository(=Dao)가 아닌 Service에 @Transactional 어노테이션을 걸었으면 Exception 발생 시 Rollback을 해줘야하는거 아닌가??


```java
@Service 
public class MyServiceImpl extends MyService { 
	@Override 
	@Transactional 
	public ResponseEntity updateBatchEnd(BatchEndUpDto dto) throws Exception { 
		batchRepository.updateBatchEnd(dto); 
		vodRepository.updateBatchEnd(dto); 
		encLogRepository.updateLogSched(EncLogUpdateDto.builder() 
			.status(dto.getStatus()) 
			.elEncSize(dto.getEncSize() != null ? Integer.parseInt(dto.getEncSize()) : 0) 
			.elPercent(dto.getPercent()) .mediaId(dto.getMediaId()) .elLog(dto.getErrMsg()) 
			.elBeginTime(dto.getElBeginTime())
			.elEndTime(dto.getElEndTime()) 
			.elLog(dto.getErrMsg()) .build()
		); 
		if(!dto.getEncType().equals("audio")) { 
			File thumbnail = new File(dto.getServerImgFilePath() + dto.getServerImgFileName()); // 잘못된 조건문으로 인해 여기서 문제 발생!!!!
			BufferedImage thumbnailImg = ImageIO.read(thumbnail); 
			ThumbEntity thumbEntity = new ThumbEntity(); 
			thumbEntity.setMediaId(dto.getMediaId()); 
			thumbEntity.setEncodingFilePath(dto.getServerImgFilePath()); 
			thumbEntity.setEncodingFileName(dto.getServerImgFileName()); 
			thumbEntity.setWidth(String.valueOf(thumbnailImg.getWidth())); 
			thumbEntity.setHeight(String.valueOf(thumbnailImg.getHeight())); 
			thumbEntity.setReprimageYn("Y"); // 스케줄러 생성 파일은 자동으로 대표 이미지 
			thumbEntity.setThumbType("A"); // 스케줄러 생성 파일이므로 'A' 
			thumbEntity.setRegDate(LocalDateTime.now()); 
			thumbEntity.setUserId(dto.getUserId()); 
			thumbEntity.setThumbTime(dto.getThumbTime()); 
			thumbRepository.addThumb(thumbEntity); 
		} 
		return ResponseEntity.status(HttpStatus.OK).body(true); 
	} 
}
```


위 코드에서 잘못된 조건 처리로 인해 예상치 못한 오류가 발생했다.

그런데 ENC_LOG 에는 업데이트가 잘 되어있다?????? 🫤

매서드 위에 @Transactional이 있는데 도대체 왜 롤백되지 않은걸까???



## @Transactional 어노테이션의 롤백

@Transactional 어노테이션은 예외처리 (try-catch)하지 않은 오류에 대해 모두 롤백처리 해주는 줄 알았는데 그게 아니었다.

Runtime Exception 또는 Error가 발생했을 때만 롤백 처리해주는게 기본이다.  
Checked Exception 은 롤백처리 해주지 않는다.

>🤔 Checked Exception이 뭐지? (프로그램이 제어할 수 없지만 개발자가 처리 가능한 예외라는데…)

![[Transactional Rollback 범위.png]]

출처 : [https://interconnection.tistory.com/122](https://interconnection.tistory.com/122)

위 그림에서 빨간색으로 묶인 부분은 UncheckedException, 나머지는 다 CheckedException인거다.  
(Error와 RuntimeException은 Unchecked Exception이고, Exception은 Checked Exception이다.)

즉, 저기 빨간 부분의 오류만 Rollback 해준다는거다.

```java
@Transactional 
public void test(TestEntity entity) throws Exception { 
	dao.save(entity); 
	throw new RuntimeException(); 
}
```

얘는 Rollback 해주는데,

```java
@Transactional 
public void test(TestEntity entity) throws Exception { 
	dao.save(entity); 
	throw new Exception(); 
}
```

얘는 안해준다..

>**JUnit** 테스트에서는 테스트 코드는 모두 Rollback 하기 때문에 **@Rollback(false)** 를 붙이고 테스트 하니 **정상적으로 되는것 같지 않았다.**

**그럼 Checked Exception은 어떻게 롤백해요?**

트랜잭션 어노테이션에서 제공되는 속성 중 **rollbackFor** 라는게 있다.  
롤백 하고자 하는 Exception을 지정할 수 있는 속성이다.


```java
@Transactional(rollbackFor = Exception.class) 
public void test(TestEntity entity) throws Exception { 
	dao.save(entity); 
	throw new Exception(); 
}
```

이렇게 rollbackFor 에 Exception.class 를 지정해주면 아까 rollback되지 않던 코드가 rollback 된다.

CheckedException 부모는 Exception 하나이기 때문에 rollbackFor에는 **Exception.class 하나만 지정**해주면 **모든 Exception에 대해 롤백 할 수 있게 된다.**

반대로 특정 오류가 나도 Rollback하고싶지 않을때는 **noRollbackFor** 속성에 예외 클래스를 지정해주면 된다.