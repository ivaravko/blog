---
layout: post
title: "Сравнение Active Record скоупов и методов класса"
date: 2013-03-28 05:40
comments: true
categories: [activerecord, rails, scopes, class methods]
footer: true

---

 [Большое количество обсуждений в интернете](https://www.google.com/search?btnG=1&pws=0&q=rails+%2B+scope+vs+class+method), что лучше использовать скоупы или методы класса, сводятся к
  "нет никакой разницы между ними" или "это дело вкуса". Но все же есть незначительные различия между ними.

## Определение скоупа

 Прежде всего, для более глубокого понимания, узнаем как скоупы используются. В Rails 3 скоуп определяется двумя способами:

{% codeblock lang:ruby %}
class Product < ActiveRecord::Base
  attr_accessible :description, :price, :status, :title

  scope :in_stock, where(status: 'in_stock')
  scope :sold, -> { where(status: 'sold') }
end
{% endcodeblock %}

 Главное отличие между ними, условие `:in_stock` инициализируется когда класс загружается впервые, в то время как `:sold` отложенно инициализируется непосредственно перед тем, как будет использован его результат. В Rails 4 первый способ устаревший, это означает, что всегда должны объявлять скоуп с вызываемым объектом как аргументом. Это позволит избежать проблем при попытке объявить скоуп с временным аргументом:

{% codeblock lang:ruby %}
class Product < ActiveRecord::Base
  scope :created_yesterday, where('created_at >= ?', 1.day.ago)
end
{% endcodeblock %}

 Но это не будет работать как ожидалось: `1.day.ago` вызывается когда класс загружается, а не каждый раз как скоуп вызывается.

## Скоупы это именно методы класса

 Внутри Active Record преобразует скоуп в метод класса. Концептуально, упрощенная реализация в Rails мастер выглядит примерно так:

{% codeblock lang:ruby %}
def self.scope(name, body)
  singleton_class.send(:define_method, name, &body)
end
{% endcodeblock %}

 Который заканчивается как метод класса с заданным именем и телом, вроде этого:

{% codeblock lang:ruby %}
def self.in_stock
  where(status: 'in_stock')
end
{% endcodeblock %}

 И именно поэтому большинство людей думают: "Почему я должен использовать скоупы, если это только синтаксический
 сахар для метода класса?" Далее несколько интересных примеров для Вас, чтобы подумать.

## Вызов скоупов всегда цепочный(chainable)

 Используем следующие сценарии: пользователи могут фильтровать продукты по статусу, упорядочивать по времени
 последнего обновления:

{% codeblock lang:ruby %}
class Product < ActiveRecord::Base
  scope :by_status, -> status { where(status: status) }
  scope :recent, -> { order("products.updated_at DESC") }
end
{% endcodeblock %}

 И можем вызывать их свободно, вроде этого:

{% codeblock lang:ruby %}
Product.by_status('sold').recent
# SELECT "products".* FROM "products" WHERE "products"."status" = 'sold'
#   ORDER BY products.updated_at DESC
{% endcodeblock %}

 Или с использование пользовательского параметра:

{% codeblock lang:ruby %}
Product.by_status(params[:status]).recent
# SELECT "products".* FROM "products" WHERE "products"."status" = 'sold'
#   ORDER BY products.updated_at DESC
{% endcodeblock %}

 До сих пор все хорошо. Теперь перенесем их в методы класса, только для сравнения:

{% codeblock lang:ruby %}
class Product < ActiveRecord::Base
  def self.by_status(status)
    where(status: status)
  end

  def self.recent
    order("products.updated_at DESC")
  end
end
{% endcodeblock %}

 Но теперь, что произойдет, если параметр `:status` равен `nil` или `blank`.

{% codeblock lang:ruby %}
Product.by_status(nil).recent
# SELECT "products".* FROM "products" WHERE "products"."status" IS NULL 
#   ORDER BY products.updated_at DESC

Product.by_status('').recent
# SELECT "products".* FROM "products" WHERE "products"."status" = '' 
#   ORDER BY products.updated_at DESC
{% endcodeblock %}

 Упс, не думаю что хотим разрешать эти запросы. Скоупы легко исправляем добавлением
 условия:

{% codeblock lang:ruby %}
scope :by_status, -> status { where(status: status) if status.present? }
{% endcodeblock %}

 Проверяем:

{% codeblock lang:ruby %}
Product.by_status(nil).recent
# SELECT "products".* FROM "products" ORDER BY products.updated_at DESC

Product.by_status('').recent
# SELECT "products".* FROM "products" ORDER BY products.updated_at DESC
{% endcodeblock %}

 Здорово. Теперь попробуем сделать то же самое с нашим любимым методом класса:

{% codeblock lang:ruby %}
class Product < ActiveRecord::Base
  def self.by_status(status)
    where(status: status) if status.present?
  end
end
{% endcodeblock %}

 Выполняем:

{% codeblock lang:ruby %}
Product.by_status('').recent
NoMethodError: undefined method `recent' for nil:NilClass
{% endcodeblock %}

 И :boomb:. Различие в том, что скоуп всегда возращает объект `ActiveRecord::Relation`, в то время как
  простоя реализация метода класса нет. Метод класса должен выглядеть следующим образом:

{% codeblock lang:ruby %}
def self.by_status(status)
  if status.present?
    where(status: status)
  else
    all
  end
end
{% endcodeblock %}

 Используем `all` для кейса `nil/blank`, в Rails 4 возвращается `ActiveRecord::Relation` (в предыдущей версии 
 возращается массив элементов из базы данных). В Rails 3.2.x нужно использовать `scoped` вместо `all`:

{% codeblock lang:ruby %}
Product.by_status('').recent
# SELECT "products".* FROM "products" ORDER BY products.updated_at DESC
{% endcodeblock %}

 Cовет: никогда не возвращать `nil` из метода класса, который должен работать как скоуп, в противном случае
 нарушаем условие вызова цепочкой (chainability) определенное в скоупе - всегда возвращать
  `ActiveRecord::Relation`.

## Скоупы расширяемые

 Пронумеруем страницы в следующем примере используя гем [kaminari](https://github.com/amatsuda/kaminari). Вызываем нужную страница:

{% codeblock lang:ruby %}
Product.page(2)
{% endcodeblock %}

 После этого, определяем сколько записей показывать на странице:

{% codeblock lang:ruby %}
Product.page(2).per(15)
{% endcodeblock %}

 Можно узнать общее количество страниц или где находимся на первой или последней странице:

{% codeblock lang:ruby %}
product = Product.page(2)
product.total_pages # => 2
product.first_page? # => false
product.last_page?  # => true
{% endcodeblock %}

 Когда создаем скоуп можем добавить специальные расширения, которые будут достпны только в нашем объекте
 при вызове этого скоупа. В случае kaminari, это только добавление скоупа страницы к `Active Record` модели
 и пологаясь на расширения скоупа добавим остальную функциональность](https://github.com/amatsuda/kaminari/blob/v0.14.1/lib/kaminari/models/active_record_model_extension.rb#L12-L17), когда будет вызвана страница. Концептуально,
 код будет выглядеть следующим образом.

{% codeblock lang:ruby %}
scope :page, -> num { # лимит + логика нумерации страниц } do
  def per(num)
    # еще логика
  end
 
  def total_pages
    # еще здесь
  end
 
  def first_page?
    # и еще
  end
 
  def last_page?
    # и так далее
  end
end
{% endcodeblock %}

 Скоуп расширения представляют собой мощную и гибкую технику в нашем наборе инcтрументов. Но, конечно, мы всегда
 можем выбрать безумный путь и получить все это используя метод класса:

{% codeblock lang:ruby %}
def self.page(num)
  scope = # some limit + offset logic here for pagination
  scope.extend PaginationExtensions
  scope
end
 
module PaginationExtensions
  def per(num)
    # еще логика
  end
 
  def total_pages
    # еще здесь
  end
 
  def first_page?
    # и еще
  end
 
  def last_page?
    # и так далее
  end
end
{% endcodeblock %}

 Это немного более многострочный код, чем при использовании скоупа, но дает тот же результат. И совет сдесь:
 выбрать то, что лучше работает для вас, но убедитесь, что вы знаете, что предлагает фреймворк, прежде чем изобретать велосипед.

## Подводя итоги

 Считаю, это важно разъяснить основные различия между скоупами и методами класса, так что Вы можите выбрать правильный инструмент для работы или инструмет более удобный. Не думаю, что действительно имеет значение использовать скоупы или методы класса, если их использовать четко и последовательно по всему приложению.

### Ссылки
* [Active Record scopes vs class methods](http://blog.plataformatec.com.br/2013/02/active-record-scopes-vs-class-methods/)
* [APIdock scope](http://apidock.com/rails/ActiveRecord/NamedScope/ClassMethods/scope)
