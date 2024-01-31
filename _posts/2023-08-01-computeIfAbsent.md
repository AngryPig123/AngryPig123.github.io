---
title: JAVA MAP<> computelfAbsent
author: Angrypig
date: 2023-08-01 15:00:00 +0800
categories: [Java]
tags: [Basic]
pin: true
math: true
mermaid: true
image:
  path: /assets/images/java/thumbnail.png
---


List<EducationTargetGroupRes> 형태의 데이터를
List에 있는 값을 기준으로 Map<String,List<String>> 형태로 만들때 사용할 수 있는 메소드


- 적용전
```java
    private Map<String, List<String>> createTargetGroupEducationTargetDivisionList() {
        List<TargetGroupEducationTargetDivision> targetGroupEducationTargetDivision = educationTargetGroupMapper.getTargetGroupEducationTargetDivision();
        Map<String, List<String>> educationTargetDivisionMap = new HashMap<>();
        targetGroupEducationTargetDivision.forEach((item) -> {
            if (!educationTargetDivisionMap.containsKey(item.getId())) {
                educationTargetDivisionMap.put(item.getId(), new ArrayList<String>());
                educationTargetDivisionMap.get(item.getId()).add(item.getEducationDivision());
            } else {
                educationTargetDivisionMap.get(item.getId()).add(item.getEducationDivision());
            }
        });
        return educationTargetDivisionMap;
    }
```


- 적용후
```java
public class Example {
    private Map<String, List<String>> getStringListMap() {
        List<TargetGroupEducationTargetDivision> targetGroupEducationTargetDivision = educationTargetGroupMapper.getTargetGroupEducationTargetDivision();
        Map<String, List<String>> educationTargetDivisionMap = new HashMap<>();

        for (TargetGroupEducationTargetDivision item : targetGroupEducationTargetDivision) {
            educationTargetDivisionMap.computeIfAbsent(item.getId(), k -> new ArrayList<>()).add(item.getGroupName());
        }

        return educationTargetDivisionMap;
    }
}
```