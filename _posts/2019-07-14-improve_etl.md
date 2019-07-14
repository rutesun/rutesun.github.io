---
layout: posts
title: ETL 파이프라인 개선하기
description: ETL 파이프라인 개선하기
image: 
category: project 
tags:
- airflow
 
toc: true
date: 2019-07-14 17:34 +0900
---
# 개요
ETL(추출, 변환 및 로드)은 다양한 원본에서 데이터를 수집하고, 비즈니스 규칙에 따라 데이터를 변환하고, 대상 데이터 저장소로 로드하는 데 사용되는 데이터 파이프라인. 

# What to solve?
기존 ETL 파이파라인은 데이터 팀에서 맡고 있었는데 데이터팀은 저희는 분석만하기로 되었어요! 라면서 업무가 이관이 되었다.
Airflow 와 Aws EMR을 주 ETL 툴로 쓰고 있었는데 이 airflow가 문제였다. 추출과 변환은 airflow로 하고 데이터의 로드는 로컬 db를 이용하게 설계되어 있었다. 문제는 airflow와 db가 같은 인스턴스에 존재하고 있다는 것. airflow 가 cpu, memory, network I/O, disk I/O 를 신나게 써서 데이터를 변환하고 db 에 load 하는 과정이 같은 인스턴스에서 발생하니 인스턴스를 아무리 scale up 해도 버텨낼 재간이 있을리가. 

렌딧에서는 data 쿼리는 모두가 가능하고 할 줄 알아야한다. 라는 모토가 있기에 [redash](https://redash.io/) 라는 툴로 모든 직원들이 자기에게 필요한 쿼리들을 직접 짜서 사용하고 있는데 한참 데이터를 로드하고 있는 database에 핵쿼리를 날리니 갈 수록 문제는 더 심해지고 있었다. 

인프라를 서로 분리하려고 하니 또 다른 문제점이 발견되었는데 airflow 안의 스크립트들의 data source, load 들의 접속 정보들과 transform 의 정보들이 하드코딩 되어있었다 또르르.

![나한테 왜 그랬어요](/assets/images/1028612757_315T0naD_59b4e7c592eb9cb1ae9dfe71580f691f926ce720.jpg)


# How to solve?

1. ETL의 인프라 분리
db는 로컬 설치형에서 aws rds 서비스로 이전하고 airflow도 새로운 인스턴스로 scaling 이 되도록 구성하기로 하였다. 
airflow가 사용하는 메세지 브로커로 redis 를 사용하고 있었는데 이것도 aws 로 이전하였다. 
다
2. datasource 들의 접속 정보가 하드코딩
각각의 접속 정보도 [Airflow Connections](https://airflow.apache.org/howto/connection/index.html) 기능을 통해서 환경변수화 하기로 결정. 반복되는 작업들도 airflow의 custom operation 과 hook 으로 다시 작성하였다.

3. database의 분리
기존에는 write 하는 db와 데이터를 뷰하는 db가 같았었는데 redash가 바라보는 data source는 read replica 를 바라보기로 해서 ETL 파이프라인에 영향을 안주도록 변경하였다. Redash 에 저장해서 사용하던 쿼리들의 source 들은 새로운 datasource id로 치환해서 end user들에게 영향이 없게 하였다.



 




