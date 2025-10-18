# Tutorial: App Lista de Tarefas com Hotwire Native

Este tutorial te guia na criação de um app simples de lista de tarefas usando Ruby on Rails e Hotwire Native para iOS e Android.

## Sumário
1. [Configuração do Rails](#1-configuração-do-rails)
2. [Modelo e Controller](#2-modelo-e-controller)
3. [Views com Turbo](#3-views-com-turbo)
4. [App iOS](#4-app-ios)
5. [App Android](#5-app-android)
6. [Testando o App](#6-testando-o-app)

---

## 1. Configuração do Rails

### Criar novo projeto Rails
```bash
rails new todo_app --database=sqlite3
cd todo_app
```

### Adicionar gems essenciais
```ruby
# Gemfile
gem 'turbo-rails'
gem 'stimulus-rails'
gem 'importmap-rails'
```

```bash
bundle install
rails turbo:install stimulus:install
```

### Configurar aplicação para mobile
```ruby
# config/application.rb
config.force_ssl = false # para desenvolvimento local
config.hosts.clear # permitir qualquer host em desenvolvimento
```

---

## 2. Modelo e Controller

### Criar modelo Task
```bash
rails generate model Task title:string completed:boolean
rails db:migrate
```

### Definir modelo
```ruby
# app/models/task.rb
class Task < ApplicationRecord
  validates :title, presence: true
  scope :completed, -> { where(completed: true) }
  scope :pending, -> { where(completed: false) }
end
```

### Controller
```ruby
# app/controllers/tasks_controller.rb
class TasksController < ApplicationController
  before_action :set_task, only: [:show, :edit, :update, :destroy, :toggle]

  def index
    @tasks = Task.all.order(:id)
    @task = Task.new
  end

  def create
    @task = Task.new(task_params)
    if @task.save
      redirect_to tasks_path, notice: 'Tarefa criada!'
    else
      @tasks = Task.all.order(:id)
      render :index, status: :unprocessable_entity
    end
  end

  def toggle
    @task.update(completed: !@task.completed)
    redirect_to tasks_path
  end

  def destroy
    @task.destroy
    redirect_to tasks_path, notice: 'Tarefa removida!'
  end

  private

  def set_task
    @task = Task.find(params[:id])
  end

  def task_params
    params.require(:task).permit(:title)
  end
end
```

### Rotas
```ruby
# config/routes.rb
Rails.application.routes.draw do
  root 'tasks#index'
  resources :tasks, only: [:index, :create, :destroy] do
    member do
      patch :toggle
    end
  end
end
```

---

## 3. Views com Turbo

### Layout principal
```erb
<!-- app/views/layouts/application.html.erb -->
<!DOCTYPE html>
<html>
  <head>
    <title>Lista de Tarefas</title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    
    <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
    <%= javascript_importmap_tags %>
    
    <!-- Metadados para Hotwire Native -->
    <meta name="turbo-native-bar-style" content="light">
    <%= content_for :head %>
  </head>

  <body>
    <main>
      <%= yield %>
    </main>
  </body>
</html>
```

### View principal (index)
```erb
<!-- app/views/tasks/index.html.erb -->
<%= content_for :title, "Minhas Tarefas" %>

<div class="container">
  <header class="header">
    <h1>📝 Lista de Tarefas</h1>
  </header>

  <!-- Formulário para nova tarefa -->
  <section class="new-task">
    <%= form_with model: @task, local: false, class: "task-form" do |form| %>
      <div class="input-group">
        <%= form.text_field :title, placeholder: "Digite uma nova tarefa...", 
                           class: "task-input", required: true %>
        <%= form.submit "Adicionar", class: "btn btn-primary" %>
      </div>
      
      <% if @task.errors.any? %>
        <div class="error">
          <%= @task.errors.full_messages.first %>
        </div>
      <% end %>
    <% end %>
  </section>

  <!-- Lista de tarefas -->
  <section class="tasks-list">
    <% if @tasks.any? %>
      <div class="tasks-summary">
        <span><%= @tasks.pending.count %> pendentes</span>
        <span><%= @tasks.completed.count %> concluídas</span>
      </div>
      
      <div id="tasks">
        <% @tasks.each do |task| %>
          <%= render 'task', task: task %>
        <% end %>
      </div>
    <% else %>
      <div class="empty-state">
        <p>🎉 Nenhuma tarefa ainda!</p>
        <p>Adicione sua primeira tarefa acima</p>
      </div>
    <% end %>
  </section>
</div>
```

### Partial da tarefa
```erb
<!-- app/views/tasks/_task.html.erb -->
<div class="task-item <%= 'completed' if task.completed %>">
  <div class="task-content">
    <%= button_to toggle_task_path(task), 
                  method: :patch, 
                  remote: true, 
                  class: "toggle-btn" do %>
      <span class="checkbox <%= 'checked' if task.completed %>">
        <%= task.completed? ? '✅' : '⭕' %>
      </span>
    <% end %>
    
    <span class="task-title"><%= task.title %></span>
  </div>
  
  <div class="task-actions">
    <%= button_to task_path(task), 
                  method: :delete, 
                  data: { 
                    confirm: "Remover '#{task.title}'?",
                    turbo_method: :delete 
                  },
                  class: "btn btn-danger" do %>
      🗑️
    <% end %>
  </div>
</div>
```

### CSS básico
```css
/* app/assets/stylesheets/application.css */
body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  margin: 0;
  padding: 20px;
  background-color: #f8f9fa;
}

.container {
  max-width: 600px;
  margin: 0 auto;
}

.header {
  text-align: center;
  margin-bottom: 30px;
}

.header h1 {
  color: #333;
  font-size: 2em;
  margin: 0;
}

.new-task {
  background: white;
  padding: 20px;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  margin-bottom: 20px;
}

.input-group {
  display: flex;
  gap: 10px;
}

.task-input {
  flex: 1;
  padding: 12px;
  border: 2px solid #e9ecef;
  border-radius: 8px;
  font-size: 16px;
}

.task-input:focus {
  outline: none;
  border-color: #007bff;
}

.btn {
  padding: 12px 20px;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  cursor: pointer;
  transition: background-color 0.2s;
}

.btn-primary {
  background-color: #007bff;
  color: white;
}

.btn-primary:hover {
  background-color: #0056b3;
}

.btn-danger {
  background-color: transparent;
  color: #dc3545;
  padding: 8px;
  font-size: 18px;
}

.tasks-summary {
  display: flex;
  justify-content: space-between;
  margin-bottom: 15px;
  font-size: 14px;
  color: #666;
}

.task-item {
  background: white;
  padding: 15px;
  margin-bottom: 10px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.task-item.completed {
  opacity: 0.6;
}

.task-content {
  display: flex;
  align-items: center;
  gap: 12px;
  flex: 1;
}

.toggle-btn {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 20px;
}

.task-title {
  font-size: 16px;
  color: #333;
}

.task-item.completed .task-title {
  text-decoration: line-through;
  color: #888;
}

.empty-state {
  text-align: center;
  padding: 40px;
  color: #666;
}

.error {
  color: #dc3545;
  margin-top: 10px;
  font-size: 14px;
}

/* Mobile friendly */
@media (max-width: 600px) {
  body {
    padding: 10px;
  }
  
  .input-group {
    flex-direction: column;
  }
  
  .task-input {
    margin-bottom: 10px;
  }
}
```

---

## 4. App iOS

### Criar projeto no Xcode
1. Abra o Xcode
2. Create new iOS App
3. Nome: "TodoList"
4. Language: Swift
5. Interface: Storyboard

### Adicionar Turbo iOS
1. File → Add Package Dependencies
2. URL: `https://github.com/hotwired/turbo-ios`
3. Add Package

### Configurar SceneDelegate
```swift
// SceneDelegate.swift
import UIKit
import Turbo

class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?
    private lazy var navigator = Navigator(session: session)
    private lazy var session = Session()

    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options connectionOptions: UIScene.ConnectionOptions) {
        guard let windowScene = scene as? UIWindowScene else { return }
        
        window = UIWindow(windowScene: windowScene)
        window?.rootViewController = navigator
        window?.makeKeyAndVisible()
        
        // URL da sua aplicação Rails (substitua pelo seu endereço)
        let startURL = URL(string: "http://localhost:3000")!
        navigator.route(url: startURL)
    }
}
```

### Configurar Navigator
```swift
// Navigator.swift
import UIKit
import Turbo

final class Navigator: UINavigationController {
    private let session: Session
    
    init(session: Session) {
        self.session = session
        super.init(nibName: nil, bundle: nil)
        self.session.delegate = self
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    func route(url: URL) {
        let visitable = VisitableViewController(url: url)
        
        if presentedViewController != nil {
            dismiss(animated: true) {
                self.pushViewController(visitable, animated: true)
            }
        } else if let top = topViewController, top is VisitableViewController {
            pushViewController(visitable, animated: true)
        } else {
            setViewControllers([visitable], animated: false)
        }
        
        session.visit(visitable)
    }
}

extension Navigator: SessionDelegate {
    func session(_ session: Session, didProposeVisit proposal: VisitProposal) {
        route(url: proposal.url)
    }
    
    func session(_ session: Session, didFailRequestForVisitable visitable: Visitable, error: Error) {
        print("Visit failed: \(error)")
    }
    
    func sessionWebViewProcessDidTerminate(_ session: Session) {
        session.reload()
    }
}
```

---

## 5. App Android

### Criar projeto Android Studio
1. New Project → Empty Activity
2. Name: "TodoList"
3. Language: Kotlin
4. API Level: 24+

### Configurar Gradle
```gradle
// app/build.gradle
dependencies {
    implementation 'dev.hotwire:turbo:7.0.1'
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.4'
}
```

### MainActivity
```kotlin
// MainActivity.kt
package com.exemplo.todolist

import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import dev.hotwire.turbo.activities.TurboActivity
import dev.hotwire.turbo.delegates.TurboActivityDelegate

class MainActivity : AppCompatActivity(), TurboActivity {
    override lateinit var delegate: TurboActivityDelegate

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // Inicializar Turbo delegate
        delegate = TurboActivityDelegate(this, R.id.turbo_view)
        
        // URL da aplicação Rails
        delegate.visit(url = "http://10.0.2.2:3000") // Para emulador Android
    }
}
```

### Layout
```xml
<!-- res/layout/activity_main.xml -->
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout 
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <dev.hotwire.turbo.views.TurboView
        android:id="@+id/turbo_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

### Permissões de Internet
```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

<application
    android:usesCleartextTraffic="true"
    ... >
```

---

## 6. Testando o App

### Executar Rails
```bash
rails server -b 0.0.0.0 -p 3000
```

### Testar iOS
1. Abra o projeto no Xcode
2. Selecione um simulador iOS
3. Cmd + R para executar
4. O app deve abrir a lista de tarefas

### Testar Android
1. Abra o projeto no Android Studio
2. Execute um emulador Android
3. Run → Run 'app'
4. O app deve abrir a lista de tarefas

### Funcionalidades esperadas
- ✅ Visualizar lista de tarefas
- ✅ Adicionar nova tarefa
- ✅ Marcar/desmarcar como concluída
- ✅ Remover tarefa
- ✅ Interface responsiva
- ✅ Navegação nativa (barra de navegação)

### Problemas comuns

**Rails não carrega no app:**
- Verifique se o Rails está rodando em `0.0.0.0:3000`
- Para iOS: use `http://localhost:3000`
- Para Android: use `http://10.0.2.2:3000`

**Layout quebrado:**
- Adicione `viewport meta` no layout
- Use CSS responsive
- Teste tamanhos de tela diferentes

**Formulários não funcionam:**
- Verifique `csrf_meta_tags`
- Use `local: false` nos forms com Turbo
- Confira rotas POST/PATCH/DELETE

---

## Próximos Passos

1. **Adicionar offline support** com Service Workers
2. **Push notifications** para lembretes
3. **Autenticação** de usuários
4. **Sincronização** com backend
5. **Bridge components** para recursos nativos (câmera, compartilhamento)

Este tutorial demonstra como criar um app móvel funcional usando apenas Rails + Hotwire Native, mantendo uma única base de código para web, iOS e Android!