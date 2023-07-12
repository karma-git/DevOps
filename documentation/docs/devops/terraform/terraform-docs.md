---
title: terraform-docs
tags:
  - terraform
---

https://developer.hashicorp.com/terraform/language

# Что это?

Это buz word-ы и заметки к ним после прочтения документации terraform, структура h1 хедеров совпадает с докой

**bold** - текстом стараюсь выделять зарезервированные слова в terraform
> мысли автора, могут быть неблагонадежными

# files
Из интересного тут увидел **override.tf**

С помощью этого файла можно перезаписать атрибут ресурса в других файлах. Это похоже на то, как работает deepmerge в helm-е. Стандартные values.yml из релиза, могут быть перезаписаны параметрами из релиза

Пример deepmerge на python https://karma-git.github.io/kb/devops/helm/local-debug/#merged-values

<details>
  <summary>Пример из доки</summary>
  
If you have a Terraform configuration `example.tf` with the following contents:

```hcl
resource "aws_instance" "web" {
  instance_type = "t2.micro"
  ami           = "ami-408c7f28"
}
```

...and you created a file `override.tf` containing the following:

```hcl
resource "aws_instance" "web" {
  ami = "foo"
}
```

Terraform will merge the latter into the former, behaving as if the original configuration had been as follows:

```hcl
resource "aws_instance" "web" {
  instance_type = "t2.micro"
  ami           = "foo"
}
```

</details>

**.terraform.lock.hcl** - как и в js package.json, лок в котором прописаны версии установленных провайдеров и модулей и их хэш суммы в пределах проекта(root module)

Рекомендуемая структура рут модуля

| File Name    | Purpose                                                |
| ------------ | ------------------------------------------------------ |
| main.tf      | Main configuration file containing resource definition |
| variables.tf | Contains variable declarations                         |
| outputs.tf   | Contains outputs from resources                        |
| provider.tf  | Contains Provider definition                           |
| terraform.tf | Configure Terraform behaviour                          |


# syntax

**Про комменты**:  `//` говорят не юзать, юзать стандартную `#`
**Комменты** в json-е, у которого строгаю структура, через ключ `"//"`
```json
{
    "name": "John",
    "//": "Ну джон ваще красавичк",
    "age": 25,
    "isStudent": true,
    "address": {
        "street": "123 Main St",
        "city": "New York",
        "country": "USA"
    },
    "grades": [85, 92, 78, 90]
}
```
**Style convention**: есть общие договоренности, которые не всегда фиксит `terraform fmt .`, например:
	- в именах ресурсах не использовать даши `-` только андерскоры `_`
	- между ресурсами 2 newline-а
	- между логически разными арграми в пределах ресурса, модуля, ставить newline

# resources
Самый важный компонент, который описывает *состояние инфраструктурного объекта*

Имеет атрибуты которые нужно определять при создании объекта, и так же из них можно читать.

Если ресурс тугой при создании - можно настроить **timeouts {}**

Про [*dependencies*](https://developer.hashicorp.com/terraform/language/resources/behavior#resource-dependencies) коротко:

- tf сам отлично разруливает dependencies за счет того, что один ресурс ссылается на другой через `<RESOURCE TYPE>.<NAME>.<ATTRIBUTE>`. Точно так же строятся зависимости между модулями - `<MODULE_NAME>.<OUTPUT_VARIABLE>`, эта история может называться *implicit dependency*
- с помощью мета аргумента **depends_on** - называется *explicit dependency*, рекомендуется только как *последняя мера*
- в полуавтоматическом режиме через **replace_triggered_by** блока **lifecycle {}**

Про `CRUD` операции над ресурсом, при изменении кода и их символы в `terraform plan`:

-  _Create_ создание ресурсов описанных в терраформе коде, но отсутствующих в **state**
  symbol `+`
- _Destroy_  visaversa, удаляются ресурсы которые в **state**, но не в терраформе коде
  symbol `-` 
- _Update in-place_ обновление ресурсов, у которых изменились аргументы - мутация
	symbol `~`
- _Destroy and re-create_ у ресурса что-то изменилось, но из-за ограничений API (скорее всего это не мутируемый объект, например как sts.spec) его нужно пересоздать
	symbol `-/+`
- _Read_ - объект просто почитают
    symbol `<=` 

## *meta-args* 
Важный момент, некоторые meta-args могут быть применимы и к другим `block {}` в terraform, например **for_each** работает в **output {}**

- **depends_on** - рассказывается чуть-чуть выше, явная зависимость между ресурсами
- **count** - array like итератор, из-за того, что обращение идет по индексам, а не по ключам, то может создавать различные конфузы, всегда лучше использовать **for_each**
- **for_each** - итератор по array или map объектам, который в итоге создает map и позволяет обращаться к результирующему объекту по ключам
  https://developer.hashicorp.com/terraform/language/expressions/for
- **provisioner**: выполняет какие-то действия - скрипты на локальной или ремоут машине
  Не декларативен и поэтому рекомендуется только как *последняя мера*
- **provider**: мета-арг `block {}`-ов, не следует путать с **provider {}** - это alias на какой-то провайдер, в случае если у нас несколько провайдеров одного типа - например мы в одном модуле описываем aws ресурсы в разных регионах
- **lifecycle**: блок с пачкой говорящих аргов/блоков управляющий жизненным циклом ресурса или другого блока

	- **create_before_destroy** - для какого-то blue-green :), эта штука плохо работает с **provisioner**
	- **prevent_destroy** - запрещает удалить, видимо что-то оч важное
	- **ignore_changes** - можно добавить определенные атрибуты объекта в исключения, чтобы не генерить его update при plan-ах
	- **replace_triggered_by** - триггерит пересоздание ресурса, можно сделать зависимость к другим ресурсам или к ресурсу пустышке - который мы можем редактировать руками
	- **precondition**, **postcondition** - можно проверять атрибуты до / после создания ресурсов и возбуждать ошибки
	  https://developer.hashicorp.com/terraform/language/expressions/custom-conditions#preconditions-and-postconditions
	  такую же работу может выполнять блок **check {}**

# data sources
Нужны для того, чтобы читать инфраструктурные объекты, обычно неописанные в терраформ коде, чтобы потом ссылаться на эти data ресурсы `<DATA_TYPE>.<NAME>.<ATTRIBUTE>`.

# provider
WIP

# Мысли автора
## provisioner
Хашикорп рекомендует вместо провиженера использовать нативные техники
	
### конфигурация уже в рантайме
	
1. *userdata* в aws, у других клаудпровайдеров тоже есть аналоги для конфигурации compute-а
2. дистрибутивые линукс имееют **cloud-init**
	
### создание артефакта ос

По мнению Хашикорп предпочтителен подход со сборкой ос через **packer**, но я думаю что это очень холиварный вопрос, проведу аналогию с контейнерами и их образами:

Если вам нужно ставить портянку софта и конфигурировать его часть какими-то статичными конфигами - собрать образ выглядит норм затеей, а какие-то мелкие вещи можно добить уже в рантайме. Негативная часть такого - поддержка образа и замена версий на свежий в computе-е

### provisioner:
Да его не рекомендуют, но ведь мы можем сделать провиженер идемпотентным с помощью того же ansible и это вроде ок