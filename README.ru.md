[![Build Status](https://travis-ci.org/aishek/activerecord-sortable.png)](https://travis-ci.org/aishek/activerecord-sortable)
[![Code Climate](https://codeclimate.com/github/aishek/activerecord-sortable.png)](https://codeclimate.com/github/aishek/activerecord-sortable)
[![Coverage Status](https://coveralls.io/repos/aishek/activerecord-sortable/badge.png)](https://coveralls.io/r/aishek/activerecord-sortable)
[![Dependency Status](https://gemnasium.com/aishek/activerecord-sortable.png)](https://gemnasium.com/aishek/activerecord-sortable)
[![Gem Version](https://badge.fury.io/rb/activerecord-sortable.png)](http://badge.fury.io/rb/activerecord-sortable)

activerecord-sortable
====================

[README in english](https://github.com/aishek/activerecord-sortable)

Этот gem позволяет интегрировать [jQuery UI Sortable](http://jqueryui.com/sortable/#default) с моделями в Rails-приложении: например, в системе управления сайтом можно сделать страницу для смены порядка товаров перетаскиванием на странице категории.

## Принцип работы

* Порядок поддерживается по значению целочисленного поля в модели, по-умолчанию `position`.
* Создаётся скоуп по имени поля, по-умолчанию `ordered_by_position_asc`.
* При удалении модели значения полей соседних по порядку моделей "сдвигаются" на единицу.
* Gem добавляет в модель метод `move_to!`, который перемещает модель на указанную позицию.
* Любые изменения значения поля порядка меняют значения полей `updated_at` и `updated_on`.
* В gem входит jQuery-плагин для изменения порядка с помощью перетаскивания (см. пример ниже).

## Пример

```ruby
# Gemfile

gem 'activerecord-sortable'
gem 'jquery-ui-rails' # если планируете drag and drop
```

```ruby
# app/models/thing.rb

class Thing < ActiveRecord::Base
  acts_as_sortable
end
```

```ruby
# db/migrate/20140512100816_create_things.rb

class CreateThings < ActiveRecord::Migration
  def change
    create_table :things do |t|
      t.integer :position, :null => false
      t.timestamps
    end

    add_index :things, [:position]
  end
end
```

```ruby
# config/routes.rb

Dummy::Application.routes.draw do
  resources :things, :only => [:index] do
    member do
      post :move
    end
  end

  root 'things#index'
end
```

```ruby
# app/controllers/things_controller.rb

class ThingsController < ApplicationController
  def index
    @things = Thing.ordered_by_position_asc
  end

  def move
    @thing = Thing.find(params[:id])
    @thing.move_to! params[:position]
  end
end
```

```html
<!-- app/views/things/index.html.erb -->

<h1>Sortable thing</h1>

<p>Use drag and drop to sort things, reload page, notice order kept.</p>

<ol data-role="activerecord_sortable">
  <%= render @things %>
</ol>
```

```html
<!-- app/views/things/_thing.html.erb -->

<!-- data-role, data-move-url, data-position – обязательные атрибуты -->
<li data-role="thing<%= thing.id %>" data-move-url="<%= move_thing_url(thing) %>" data-position="<%= thing.position %>">
  <h2>Thing <%= thing.id %></h2>
</li>
```

```js
// app/views/things/move.js.erb

var node = $('*[data-role="thing<%= @thing.id %>"]');
var new_node_html = '<%= j render @thing %>';

node.replaceWith(new_node_html);
```

```js
// app/assets/javascripts/application.js

//= require jquery
//= require jquery_ujs
//= require sortable

//= require jquery-ui/sortable

$(document).ready(function(){
  $('*[data-role=activerecord_sortable]').activerecord_sortable();
});
```

Смотрите также [код тестового приложения](https://github.com/aishek/activerecord-sortable/tree/master/spec/dummy).

## Настройки
```ruby
class Thing < ActiveRecord::Base
  acts_as_sortable do |config|
    # внутри какого набора объектов поддерживать порядок
    # опция полезна при использовании STI, например,
    # чтобы управлять порядком в рамках одного STI-класса, укажите
    # config[:relation] = ->(instance) {instance.class.base_class)}
    #
    # по-умолчанию – внутри всех эземпляров модели
    config[:relation] = ->(instance) {instance.class}

    # добавлять ли в конец по порядку новые экземпляры моделей,
    # по-умолчанию – вставлять в начало
    config[:append] = false

    # используемое целочисленное поле в модели
    # по-умолчанию – position
    config[:position_column] = :position
  end
end
```

## События Javascript

В примере:

```js
// app/assets/javascripts/application.js

//= require jquery
//= require jquery_ujs
//= require sortable

//= require jquery-ui/sortable

$(document).ready(function(){
  $('*[data-role=activerecord_sortable]').activerecord_sortable();
});
```

`$('*[data-role=activerecord_sortable]')` бросает события:

* `sortable:start` – запрос об изменении позиции отправлен на сервер
* `sortable:stop` – сервер ответил на запрос об изменении позиции
* `sortable:sort_success` – сервер ответил успехом на запрос об изменении позиции
* `sortable:sort_error` – сервер ответил ошибкой на запрос об изменении позиции

Перед отправкой запроса на сервер функциональность jQuery UI Sortable выключается, после получения ответа от сервера включается обратно.

## Как добавить функциональность activerecord-sortable в существующую модель

```ruby
# Gemfile

gem 'activerecord-sortable'
```

```ruby
# db/migrate/20140525112125_add_position_to_items.rb

class AddPositionToItems < ActiveRecord::Migration
  def up
    add_column :items, :position, :integer

    # не забудьте задать порядок
    Item.order('id desc').each.with_index do |item, position|
      item.update_attribute :position, position
    end

    change_column :items, :position, :integer, :null => false

    add_index :items, [:position]
  end

  def down
    remove_column :items, :position, :integer
  end
end
```

```ruby
# app/models/item.rb

class Item < ActiveRecord::Base
  acts_as_sortable
end
```

## Функциональность activerecord-sortable для вложенных моделей

```ruby
# app/models/parent.rb

class Parent < ActiveRecord::Base
  has_many :children

  accepts_nested_attributes_for :children
end
```

```ruby
# app/models/child.rb

class Child < ActiveRecord::Base
  acts_as_sortable do |config|
    config[:relation] = ->(instance) {instance.parent.children}
  end

  belongs_to :parent
end
```

```ruby
# app/controllers/parents_controller.rb

class ParentsController < ApplicationController
  def new
    @parent = Parent.new
    3.times do |position|
      @parent.children.build(
        :name => "Child #{position}",
        :position => position # не забыть указать
      )
    end
  end

  def create
    @parent = Parent.new parent_params
    if @parent.save
      redirect_to parent_path(@parent)
    else
      render :new
    end
  end

  def show
    @parent = Parent.find(params[:id])
  end


  private

  def parent_params
    params.require(:parent).permit(:children_attributes => [:name, :position])
  end
end
```

```html
<!-- app/views/parents/new.html.erb -->

<h1>New parent</h1>

<%= render :partial => 'form' %>
```

```html
<!-- app/views/parents/_form.html.erb -->

<%= form_for @parent do |f| %>
  <fieldset>
    <legend>Children</legend>

    <ol data-role="activerecord_sortable">
      <%= f.fields_for :children do |ff| %>
        <!-- оба атрибута обязательны -->
        <li data-role="child<%= ff.object.to_s %>" data-position="<%= ff.object.position %>">
          <%= ff.object.name %>

          <!-- поле для передачи позиции в контроллер -->
          <%= ff.hidden_field :position, :data => { :role => 'position' } %>
        </li>
      <% end %>
    </ol>
  </fieldset>

  <%= f.submit %>
<% end %>
```

## Патчи и пулл-реквесты

* Сделайте форк.
* Внесите изменения.
* Сделайте пулл-реквест. Ваши изменения в отдельной ветке принесут плюс в карму :)

## Лицензия

activerecord-sortable является бесплатным ПО, подробности в файле [LICENSE](https://github.com/aishek/activerecord-sortable/LICENSE).

## Авторы

Gem activerecord-sortable поддерживается [Цифрономикой](http://cifronomika.ru/).

Авторы:

* [Александр Борисов](https://github.com/aishek)
* [Кирилл Храпков](https://github.com/cubbiu)
