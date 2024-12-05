# 2024년 12월 5일 (목)
> 원자적 변수

## 개요
자바에서 Stream을 통해 파이프라인을 만들다가 외부 데이터를 수정하려 하면 컴파일이 안되고, 원자적 변수를 사용하라고 한다.

예를 들어, 1부터 10까지의 수를 Stream을 통해 더해보자. <del><i>(매우 잘못된 코드)</i></del>

```java
int sum = 0;
IntStream.rangeClosed(1, 10).forEach((x) -> sum += x);
```

이 코드는 컴파일이 불가능하다.

물론 아래와 같이 reduce 혹은 sum와 같이 고차 함수를 사용하면 해결할 수 있다.

```java
int sum = IntStream.rangeClosed(1, 10).reduce((acc, x) -> acc = acc + x);
```

하지만 나의 경우는 경우가 달랐는데, 스프링에서 S3로 올라간 사진 중, 실질적으로 사용되지 않는 파일을 삭제하고 로깅하는 Task가 있다.

```java
@PostConstruct
@Scheduled(cron = "0 0 0 * * *")
private void run() {

    Set<String> use = Stream.of(
            activityRepository.findAll().stream().flatMap(activity -> activity.getImages().stream()),
            historyRepository.findAll().stream().map(History::getImage),
            projectRepository.findAll().stream().map(Project::getImage))
        .flatMap(s -> s)
        .collect(Collectors.toSet());

    s3Service.files("program").stream()
        .filter(file -> System.currentTimeMillis() - file.getLastModified().getTime() > 1000 * 60 * 30)
        .map(S3ObjectSummary::getKey)
        .filter(file -> !use.contains(file))
        .peek((file) -> size.getAndAdd(1))
        .map(s3Service::urlToKey)
        .forEach(s3Service::deleteFile);
}
```

하단 `s3Service.files("program")`의 코드에서 stream을 통해 데이터를 처리하고 삭제한다.

그 후 삭제된 개수를 로깅해야 하는데, forEach 후에는 이미 stream이 닫혀 사용할 수 없다.

그래서 peek을 사용하는데, peek에서 외부 데이터를 수정할 수 없었다. 아래는 오류 코드이다.

```java
@PostConstruct
@Scheduled(cron = "0 0 0 * * *")
private void run() {
	
	int amount = 0;

    s3Service.files("program").stream()
        // ...
        .peek(x -> amount++)
        .map(s3Service::urlToKey)
        .forEach(s3Service::deleteFile);

	log.info("Purge Unused S3 Resource. (amount: {})", amount);
}
```

그래서 자바에서 원자적 변수 AtomicInteger을 사용하여 해결한다.

```java
@PostConstruct
@Scheduled(cron = "0 0 0 * * *")
private void run() {
	
	AtomicInteger amount = new AtomicInteger(0);

    s3Service.files("program").stream()
        // ...
        .peek(x -> amount.getAndAdd())
        .map(s3Service::urlToKey)
        .forEach(s3Service::deleteFile);

	log.info("Purge Unused S3 Resource. (amount: {})", amount.get());
}
```

## 원자적 변수 (Atomic Variable)

일반적으로 자바에서 동시성 문제를 해결하기 위해서는 synchronized 키워드를 사용하여 lock을 건다.

하지만, synchronized는 blocking 기반이기 때문에, 멀티 스레드 환경에서 성능이 매우 떨어진다.

이러한 문제점을 해결하기 위해 non-blocking 기반의 원자적 변수가 나오게 되었다.

## CAS Algorithm

CAS는 Compare And Swap이다.

주된 로직은 파라미터로 기존 값과 변경할 값을 받고, 기존 값이 현재 메모리가 가지고 있는 값과 같다면 변경할 값으로 수정하고 true를 반환한다.

반면에 기존 값이 현재 메모리가 가지고 있는 값과 다르다면 값을 반영하지 않고 false를 반영한다.

만약 부모 함수에서 false를 받았다면, 다시 값을 읽은 후, 요청을 보내면 된다.