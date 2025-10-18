# Tutorial: App ToDo com Hotwire Native

## Introdução e Objetivos

### O que vamos construir
Neste tutorial, você criará um aplicativo de lista de tarefas (ToDo) usando Hotwire Native. O resultado final será:

- **Interface web** funcionando no navegador
- **App iOS nativo** carregando a mesma interface
- **App Android nativo** carregando a mesma interface

Tudo usando uma única base de código Rails com Turbo e Stimulus.

### Funcionalidades
- Criar, editar e deletar tarefas
- Marcar tarefas como concluídas
- Interface responsiva e nativa
- Atualizações em tempo real

---

## Pré-requisitos e Ambiente

### Ferramentas Necessárias
- **Ruby 3.0+** e **Rails 7.0+**
- **Xcode** (para iOS)
- **Android Studio** (para Android)
- **Navegador moderno** para testes web

### Alternativa Docker
Se você usa Docker, certifique-se de ter containers configurados:

```bash path=null start=null
# Verificar containers ativos
docker ps

# Os comandos Rails serão executados como:
docker compose run --rm web bundle exec rails [comando]
```

### Banco de Dados
- **Desenvolvimento**: SQLite (padrão do Rails)
- **Produção**: PostgreSQL recomendado
- **HTTPS obrigatório** em produção para apps nativos

---

## 1. Criação do Projeto Rails

### Setup Local

```bash path=null start=null
# Criar novo projeto Rails com Importmap
bundle exec rails new todo_native -j importmap --css=bootstrap

cd todo_native

# Setup inicial do banco
bundle exec rails db:setup

# Testar servidor
bundle exec rails server
```

### Setup com Docker

```bash path=null start=null
# Verificar containers
docker ps

# Criar projeto
docker compose run --rm web bundle exec rails new todo_native -j importmap --css=bootstrap

cd todo_native

# Setup do banco
docker compose run --rm web bundle exec rails db:setup

# Testar servidor
docker compose run --rm web bundle exec rails server
```

### Verificar Configuração Hotwire

O arquivo `config/importmap.rb` deve ter:

```ruby path=null start=null
# config/importmap.rb
pin "application"
pin "@hotwired/turbo-rails", to: "turbo.min.js"
pin "@hotwired/stimulus", to: "stimulus.min.js"
pin "@hotwired/stimulus-loading", to: "stimulus-loading.js"
pin_all_from "app/javascript/controllers", under: "controllers"
```

E o arquivo `app/javascript/controllers/application.js`:

```javascript path=null start=null
// app/javascript/controllers/application.js
import { Application } from "@hotwired/stimulus"

const application = Application.start()
application.debug = false
window.Stimulus = application

export { application }
```

---

## 2. Modelagem e CRUD de Task

### Gerar Scaffold

```bash path=null start=null
# Comando local
bundle exec rails g scaffold Task title:string done:boolean

# Comando Docker
docker compose run --rm web bundle exec rails g scaffold Task title:string done:boolean
```

### Ajustar Migration (Strong Migrations)

Edite a migration gerada para adicionar default seguro:

```ruby path=null start=null
# db/migrate/xxx_create_tasks.rb
class CreateTasks < ActiveRecord::Migration[7.0]
  def change
    create_table :tasks do |t|
      t.string :title, null: false
      t.boolean :done, default: false, null: false
      
      t.timestamps
    end
    
    add_index :tasks, :done
    add_index :tasks, :created_at
  end
end
```

### Executar Migration

```bash path=null start=null
# Local
bundle exec rails db:migrate

# Docker
docker compose run --rm web bundle exec rails db:migrate
```

### Configurar Rota Root

```ruby path=null start=null
# config/routes.rb
Rails.application.routes.draw do
  resources :tasks do
    member do
      patch :toggle
    end
  end
  
  root 'tasks#index'
end
```

---

## 3. Ajustar Controller para Turbo

### TasksController Otimizado

```ruby path=null start=null
# app/controllers/tasks_controller.rb
class TasksController < ApplicationController
  before_action :set_task, only: [:show, :edit, :update, :destroy, :toggle]

  def index
    @tasks = Task.order(:created_at)
    @task = Task.new
  end

  def create
    @task = Task.new(task_params)

    if @task.save
      respond_to do |format|
        format.turbo_stream
        format.html { redirect_to root_path, notice: 'Tarefa criada.' }
      end
    else
      respond_to do |format|
        format.turbo_stream { render :new, status: :unprocessable_entity }
        format.html { render :new, status: :unprocessable_entity }
      end
    end
  end

  def update
    if @task.update(task_params)
      respond_to do |format|
        format.turbo_stream
        format.html { redirect_to root_path, notice: 'Tarefa atualizada.' }
      end
    else
      respond_to do |format|
        format.turbo_stream { render :edit, status: :unprocessable_entity }
        format.html { render :edit, status: :unprocessable_entity }
      end
    end
  end

  def destroy
    @task.destroy
    
    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to root_path, notice: 'Tarefa removida.' }
    end
  end

  def toggle
    @task.update(done: !@task.done)
    
    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to root_path }
    end
  end

  private

  def set_task
    @task = Task.find(params[:id])
  end

  def task_params
    params.require(:task).permit(:title, :done)
  end
end
```

---

## 4. Views com Turbo Frames

### Index Principal

```erb path=null start=null
<!-- app/views/tasks/index.html.erb -->
<div class="container mt-4">
  <h1>Minhas Tarefas</h1>
  
  <!-- Formulário de nova tarefa -->
  <%= turbo_frame_tag "new_task" do %>
    <%= render "form", task: @task %>
  <% end %>
  
  <!-- Lista de tarefas -->
  <%= turbo_frame_tag "tasks" do %>
    <%= render @tasks %>
  <% end %>
  
  <!-- Para broadcasts em tempo real (opcional) -->
  <%= turbo_stream_from "tasks" %>
</div>
```

### Partial da Tarefa (HTML Mínimo)

```erb path=null start=null
<!-- app/views/tasks/_task.html.erb -->
<div id="<%= dom_id(task) %>" class="card mb-2">
  <div class="card-body d-flex justify-content-between align-items-center">
    <div>
      <h5 class="<%= 'text-decoration-line-through' if task.done %>">
        <%= task.title %>
      </h5>
      <small class="text-muted">
        Status: <%= task.done? ? "Concluída" : "Pendente" %>
      </small>
    </div>
    
    <div class="btn-group">
      <%= button_to toggle_task_path(task), 
                    method: :patch, 
                    remote: true,
                    class: "btn #{'btn-success' if task.done?} #{'btn-outline-success' unless task.done?}" do %>
        <%= task.done? ? "✓" : "○" %>
      <% end %>
      
      <%= link_to "Editar", edit_task_path(task), 
                  class: "btn btn-outline-primary" %>
      
      <%= button_to "Excluir", task_path(task), 
                    method: :delete, 
                    remote: true,
                    class: "btn btn-outline-danger",
                    data: { confirm: "Tem certeza?" } %>
    </div>
  </div>
</div>
```

### Formulário de Nova Tarefa

```erb path=null start=null
<!-- app/views/tasks/_form.html.erb -->
<%= form_with model: task, 
              local: false, 
              class: "card mb-4" do |form| %>
  <div class="card-body">
    <% if task.errors.any? %>
      <div class="alert alert-danger">
        <ul class="mb-0">
          <% task.errors.full_messages.each do |message| %>
            <li><%= message %></li>
          <% end %>
        </ul>
      </div>
    <% end %>

    <div class="mb-3">
      <%= form.text_field :title, 
                          placeholder: "Nova tarefa...", 
                          class: "form-control",
                          required: true %>
    </div>

    <div class="d-flex gap-2">
      <%= form.submit "Adicionar", class: "btn btn-primary" %>
      <% if task.persisted? %>
        <%= link_to "Cancelar", root_path, class: "btn btn-secondary" %>
      <% end %>
    </div>
  </div>
<% end %>
```

---

## 5. Turbo Streams para Atualizações

### Criar Tarefa

```erb path=null start=null
<!-- app/views/tasks/create.turbo_stream.erb -->
<%= turbo_stream.prepend "tasks" do %>
  <%= render @task %>
<% end %>

<%= turbo_stream.replace "new_task" do %>
  <%= render "form", task: Task.new %>
<% end %>
```

### Atualizar Tarefa

```erb path=null start=null
<!-- app/views/tasks/update.turbo_stream.erb -->
<%= turbo_stream.replace dom_id(@task) do %>
  <%= render @task %>
<% end %>

<%= turbo_stream.replace "new_task" do %>
  <%= render "form", task: Task.new %>
<% end %>
```

### Toggle Tarefa

```erb path=null start=null
<!-- app/views/tasks/toggle.turbo_stream.erb -->
<%= turbo_stream.replace dom_id(@task) do %>
  <%= render @task %>
<% end %>
```

### Deletar Tarefa

```erb path=null start=null
<!-- app/views/tasks/destroy.turbo_stream.erb -->
<%= turbo_stream.remove dom_id(@task) %>
```

---

## 6. Model com Broadcasts (Opcional)

Para atualizações em tempo real entre dispositivos:

```ruby path=null start=null
# app/models/task.rb
class Task < ApplicationRecord
  validates :title, presence: true
  
  # Broadcasts para tempo real (opcional)
  after_create_commit { broadcast_prepend_to "tasks", partial: "tasks/task" }
  after_update_commit { broadcast_replace_to "tasks", partial: "tasks/task" }
  after_destroy_commit { broadcast_remove_to "tasks" }
end
```

---

## 7. Integração iOS (Turbo iOS)

### Criar Projeto Xcode

1. Abra o Xcode
2. Create a new project → iOS → App
3. Nome: "TodoNative"
4. Language: Swift
5. Interface: Storyboard

### Adicionar Turbo via Swift Package Manager

1. Em Xcode: File → Add Package Dependencies
2. URL: `https://github.com/hotwired/turbo-ios`
3. Adicionar à target principal

### Implementar Session e Navigator

```swift path=null start=null
// ViewController.swift
import UIKit
import Turbo

class ViewController: UIViewController {
    private lazy var session = Session()
    private lazy var navigator = TurboNavigator(session: session)
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        session.delegate = self
        navigator.delegate = self
        
        // URL do servidor Rails (desenvolvimento)
        visit(URL(string: "http://localhost:3000")!)
    }
    
    private func visit(_ url: URL) {
        let visitable = VisitableViewController(url: url)
        navigator.route(visitable)
    }
}

extension ViewController: SessionDelegate {
    func session(_ session: Session, didProposeVisit proposal: VisitProposal) {
        visit(proposal.url)
    }
    
    func session(_ session: Session, didFailRequestForVisitable visitable: Visitable, error: Error) {
        print("Request failed: \(error)")
    }
}

extension ViewController: TurboNavigatorDelegate {
    func navigator(_ navigator: TurboNavigator, didProposeVisit proposal: VisitProposal) {
        visit(proposal.url)
    }
}
```

### Configurar ATS (App Transport Security)

No arquivo `Info.plist`, adicione (apenas para desenvolvimento):

```xml path=null start=null
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSExceptionDomains</key>
    <dict>
        <key>localhost</key>
        <dict>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/>
        </dict>
    </dict>
</dict>
```

**⚠️ Em produção, use apenas HTTPS e remova essa configuração.**

---

## 8. Integração Android (Turbo Android)

### Criar Projeto Android Studio

1. Abra Android Studio
2. Create New Project → Empty Activity
3. Nome: "TodoNative"
4. Language: Kotlin

### Adicionar Dependência Turbo Android

No arquivo `build.gradle` (Module: app):

```kotlin path=null start=null
dependencies {
    implementation 'dev.hotwire:turbo-android:7.0.0'
    // outras dependências...
}
```

### Implementar Activity Principal

```kotlin path=null start=null
// MainActivity.kt
package com.example.todonative

import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import dev.hotwire.turbo.activities.TurboActivity
import dev.hotwire.turbo.delegates.TurboActivityDelegate

class MainActivity : AppCompatActivity(), TurboActivityDelegate {
    private lateinit var turboActivityDelegate: TurboActivityDelegate

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        turboActivityDelegate = TurboActivityDelegate(this, R.id.turbo_view)
        
        // URL do servidor Rails (desenvolvimento - emulador Android)
        turboActivityDelegate.visit("http://10.0.2.2:3000")
    }

    override fun onTurboVisit() {
        // Implementar se necessário
    }
}
```

### Layout Principal

```xml path=null start=null
<!-- res/layout/activity_main.xml -->
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <dev.hotwire.turbo.views.TurboWebView
        android:id="@+id/turbo_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</LinearLayout>
```

### Configurar Network Security (Desenvolvimento)

Crie `res/xml/network_security_config.xml`:

```xml path=null start=null
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">10.0.2.2</domain>
    </domain-config>
</network-security-config>
```

E adicione no `AndroidManifest.xml`:

```xml path=null start=null
<application
    android:networkSecurityConfig="@xml/network_security_config"
    ...>
```

**⚠️ Em produção, use apenas HTTPS e remova essa configuração.**

---

## 9. Testar o App

### Teste Web

```bash path=null start=null
# Iniciar servidor
bundle exec rails server

# Acessar: http://localhost:3000
```

### Teste iOS

1. Rode o servidor Rails: `bundle exec rails server`
2. Execute o app iOS no simulador
3. Deve carregar a interface web dentro do app

### Teste Android

1. Rode o servidor Rails: `bundle exec rails server`
2. Execute o app Android no emulador
3. Use a URL `http://10.0.2.2:3000` (IP do host no emulador)

---

## 10. Autenticação (Opcional)

### Adicionar Devise

```bash path=null start=null
# Adicionar no Gemfile
echo "gem 'devise'" >> Gemfile

# Local
bundle install
bundle exec rails generate devise:install
bundle exec rails generate devise User
bundle exec rails db:migrate

# Docker
docker compose run --rm web bundle install
docker compose run --rm web bundle exec rails generate devise:install
docker compose run --rm web bundle exec rails generate devise User
docker compose run --rm web bundle exec rails db:migrate
```

### Associar Tasks a Users

```bash path=null start=null
# Gerar migration
bundle exec rails g migration AddUserToTasks user:belongs_to

# Docker
docker compose run --rm web bundle exec rails g migration AddUserToTasks user:belongs_to
```

```ruby path=null start=null
# Migration gerada (ajustar se necessário)
class AddUserToTasks < ActiveRecord::Migration[7.0]
  def change
    add_reference :tasks, :user, null: false, foreign_key: true
  end
end
```

### Ajustar Models

```ruby path=null start=null
# app/models/task.rb
class Task < ApplicationRecord
  belongs_to :user
  validates :title, presence: true
end

# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable
         
  has_many :tasks, dependent: :destroy
end
```

### Proteger Controller

```ruby path=null start=null
# app/controllers/tasks_controller.rb
class TasksController < ApplicationController
  before_action :authenticate_user!
  before_action :set_task, only: [:show, :edit, :update, :destroy, :toggle]

  def index
    @tasks = current_user.tasks.order(:created_at)
    @task = current_user.tasks.build
  end

  def create
    @task = current_user.tasks.build(task_params)
    # resto do código...
  end

  private

  def set_task
    @task = current_user.tasks.find(params[:id])
  end
end
```

---

## 11. Deploy e Publicação

### Deploy Backend (Exemplo: Render)

1. **Criar conta no Render**
2. **Conectar repositório GitHub**
3. **Configurar variáveis de ambiente**:
   - `RAILS_MASTER_KEY`
   - `DATABASE_URL` (PostgreSQL)
4. **Build e Deploy**:

```bash path=null start=null
# Comandos que o Render executará
bundle install
bundle exec rails assets:precompile
bundle exec rails db:migrate
```

### Preparar iOS para Produção

1. **Atualizar URL de produção**:
```swift path=null start=null
// Trocar localhost pela URL de produção
visit(URL(string: "https://seu-app.render.com")!)
```

2. **Remover configuração ATS de desenvolvimento**
3. **Configurar assinatura de código**
4. **Enviar para TestFlight**

### Preparar Android para Produção

1. **Atualizar URL de produção**:
```kotlin path=null start=null
// Trocar IP local pela URL de produção
turboActivityDelegate.visit("https://seu-app.render.com")
```

2. **Remover Network Security Config de desenvolvimento**
3. **Gerar APK de release**
4. **Publicar na Play Console**

---

## 12. Troubleshooting

### Problemas Comuns

#### Cookies não persistem
- **Causa**: Domínios diferentes entre web e native
- **Solução**: Use o mesmo domínio HTTPS

#### Navegação não funciona
- **iOS**: Verificar implementação do `SessionDelegate`
- **Android**: Verificar `TurboActivityDelegate`

#### Cleartext HTTP bloqueado
- **Causa**: Política de segurança em produção
- **Solução**: Usar HTTPS sempre

#### Turbo Streams não funcionam
- **Causa**: JavaScript ou Action Cable não configurado
- **Solução**: Verificar `importmap.rb` e broadcasts

### Comandos de Debug

```bash path=null start=null
# Verificar rotas
bundle exec rails routes | grep tasks

# Verificar assets
bundle exec rails assets:clobber
bundle exec rails assets:precompile

# Logs em desenvolvimento
tail -f log/development.log
```

---

## Checklist Final

### ✅ Funcionalidades Básicas
- [ ] CRUD de tarefas funciona no navegador
- [ ] Toggle de status funciona
- [ ] Interface responsiva

### ✅ Apps Nativos
- [ ] App iOS carrega interface corretamente
- [ ] App Android carrega interface corretamente
- [ ] Navegação nativa funciona
- [ ] Formulários funcionam nos apps

### ✅ Produção
- [ ] Backend deployado em HTTPS
- [ ] Apps configurados para produção
- [ ] Autenticação funciona (se implementada)
- [ ] Performance aceitável

### ✅ Publicação
- [ ] iOS: TestFlight funcionando
- [ ] Android: Play Console configurado
- [ ] Store listings preparadas

---

## Conclusão

Parabéns! Você criou um app completo usando Hotwire Native. Com uma única base de código Rails, você tem:

- Interface web responsiva
- App iOS nativo
- App Android nativo
- Atualizações em tempo real
- Autenticação funcional

**Próximos passos**:
1. Adicione mais funcionalidades (categorias, prazos)
2. Melhore o design visual
3. Implemente notificações push
4. Publique nas stores

Lembre-se: mantenha simples, teste constantemente e publique incrementalmente!