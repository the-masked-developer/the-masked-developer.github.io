---
title: 업무시간에 몰래 작성해보는 springboot jpa entity와 dto
date: 2021-12-15 17:50:00
math: true
toc: true
index_img:
categories:
    - java
    - springboot
tags:
    - java
    - springboot
author: moon
---
### JPA, JpaRepository 소개

---

JPA란?  
+ Java ORM 기술 표준으로 사용하는 인터페이스 모음  

JpaRepository란?  
+ Spring, SpringBoot Framework에서 제공하는 인터페이스
    - QueryByExampleExecutor와 PagingAndSortingRepository를 상속함
    - 기본적인 수준의 검색 메서드가 정의되어 있어서 override하여 사용 가능
    
```java
public interface JpaRepository<T, ID> extends PagingAndSortingRepository<T, ID>, QueryByExampleExecutor<T> {

	/*
	 * (non-Javadoc)
	 * @see org.springframework.data.repository.CrudRepository#findAll()
	 */
	@Override
	List<T> findAll();

	/*
	 * (non-Javadoc)
	 * @see org.springframework.data.repository.PagingAndSortingRepository#findAll(org.springframework.data.domain.Sort)
	 */
	@Override
	List<T> findAll(Sort sort);

	/*
	 * (non-Javadoc)
	 * @see org.springframework.data.repository.CrudRepository#findAll(java.lang.Iterable)
	 */
	@Override
	List<T> findAllById(Iterable<ID> ids);
	
	...
```

### Entity와 Dto

---
다음과 같은 테이블 Coupon 있다고 하자

| Column        | Type              | Default Value     | Nullable | 그 외                            | 
| ------------- | ----------------- | ----------------- | -------- | ------------------------------  |
| no            | int               |                   | NO       | Primary Key, Auto Increment     |
| coupon_no     | varchar           |                   | NO       | 쿠폰번호                          |
| use_date      | timestamp         |                   | YES      | 사용일자                          |
| use_flag      | enum('Y','N')     | N                 | NO       | 사용여부                          |
| discount      | int               | 0                 | NO       | 할인가격                          |
| description   | varchar           |                   | YES      | 쿠폰 설명                         |
| discount_type | enum('P', 'W')    | W                 | NO       | 할인 방식 ('P' : 퍼센티지, 'W' : 원) |
| create_date   | timestamp         | CURRENT_TIMESTAMP | NO       | Default Generated 생성 일자       |

Entity는 해당 테이블과 매핑하여 사용  
Dto는 테이블에서 받은 Entity를 다른 layer로 전송할때 사용 (Data Transfer Object)

entity와 dto는 다음과 같이 만들 수 있다.

ENTITY
```java
package entity;

import enums.DiscountType;
import enums.YorN;
import lombok.*;
import org.hibernate.annotations.CreationTimestamp;
import javax.persistence.*;
import java.time.LocalDateTime;

@ToString
@Builder
@AllArgsConstructor
@NoArgsConstructor
@Getter
@Setter
@Entity
@Table(name="coupon")
public class Coupon {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "no")
    private int no;

    @Column(name = "coupon_no")
    private String couponNo;

    @Column(name = "use_date")
    @Builder.Default
    private LocalDateTime useDate = null;

    @Enumerated(EnumType.STRING)    // EnumType은 ORDINAL과 STRING이 있음     
    @Builder.Default                // Builder로 생성 시 default 값 설정
    @Column(name = "use_flag")
    private YorN useFlag = YorN.N; // 설정될 default값 ('N')

    @Column(name = "discount")
    @Builder.Default
    private int discount = 0;

    @Column(name = "description", nullable = true)
    private String description;

    @Enumerated(EnumType.STRING)
    @Column(name = "discount_type")
    @Builder.Default
    private DiscountType discountType = DiscountType.W;

    @CreationTimestamp
    @Column(name= "create_date", columnDefinition = "TIMESTAMP")
    private LocalDateTime createDate;
}
```

DTO
```java
package dto;

import enums.DiscountType;
import enums.YorN;
import entity.Coupon;
import lombok.*;

import java.time.LocalDateTime;

@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
public class CouponDto {
    private int no;
    private String couponNo;
    private LocalDateTime useDate;
    private YorN useFlag;
    private int discountPrice;
    private DiscountType discountType;
    private String description;
    private LocalDateTime createDate;

    @Builder(builderMethodName = "CouponDtoBuilder")
    public CouponDto(Coupon coupon){
        no = coupon.getNo();
        couponNo = coupon.getCouponNo();
        useDate = coupon.getUseDate();
        useFlag = coupon.getUseFlag();
        discountPrice = coupon.getDiscount();
        discountType = coupon.getDiscountType();
        description = coupon.getDescription();
        createDate = coupon.getCreateDate();
    }
}
```

ENUM
```java
package enums;

import com.fasterxml.jackson.annotation.JsonValue;
import java.util.Locale;

public enum DiscountType {
    P, W;

    @JsonValue
    public String getStatus() { return this.name().toUpperCase(Locale.ROOT); }
}

public enum YorN {
    Y, N;

    @JsonValue
    public String getStatus() { return this.name().toUpperCase(Locale.ROOT); }
}
```

@Enumerated(EnumType)의 경우, ORDINAL과 STRING이 존재
+ ORDINAL - P : 1, W : 2 ... enum의 순서가 DB에 저장됨
+ STRING - P, W ... enum의 문자열 값이 DB에 저장됨


JpaRepository를 상속하는 CouponRepository를 다음과 같이 생성하고

```java
package repository;

import entity.Coupon;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface CouponRepository extends JpaRepository<Coupon, Integer> {
    Coupon findCouponByNo(int no);
}

```

Service Layer에서는 다음과 같이 DTO로 사용할 수 있다.
```java
package service;

import dto.CouponDto;
import entity.Coupon;
import repository.CouponRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class CouponService {

    private final CouponRepository couponRepository;

    public CouponDto getCouponInfoByNo(int no) {
        Coupon coupon = couponRepository.findCouponByNo(no);

        CouponDto couponDto = CouponDto.CouponDtoBuilder().coupon(coupon).build();

        // Do Something

        return couponDto;
    }
}

```
