# Manual Hotwire Native

## Sumário

### O que é Hotwire Native
O Hotwire Native permite criar aplicativos móveis nativos (iOS e Android) que carregam interfaces web construídas com Rails, Turbo e Stimulus. Em vez de desenvolver duas interfaces separadas, você cria uma única interface web que funciona tanto no navegador quanto nos apps nativos.

### Para quem é este guia
Este manual é para desenvolvedores Rails que querem criar apps nativos reutilizando suas habilidades web. Você deve conhecer o básico de Rails, HTML e ter interesse em publicar apps na App Store e Play Store.

---

## Como o Hotwire Native Funciona

### Visão Geral da Arquitetura

O Hotwire Native funciona através desta arquitetura:

```text
App Nativo (iOS/Android)
    ↓
WebView com Turbo Native
    ↓ HTTP Request
Servidor Rails
    ↓ Response
HTML + Turbo Streams/Frames
    ↓
Atualização Instantânea no App
```

**Componentes principais:**
- **Turbo Drive**: Navegação rápida sem recarregar páginas
- **Turbo Frames**: Atualização de partes específicas da tela
- **Turbo Streams**: Atualizações em tempo real via WebSocket
- **Stimulus**: JavaScript mínimo para micro-interações

### Como o Servidor Rails Responde

O servidor Rails gera HTML normal e responde tanto para navegadores web quanto para apps nativos. O mesmo controlador pode servir ambos:

```ruby path=null start=null
class TasksController < ApplicationController
  def create
    @task = Task.new(task_params)
    
    if @task.save
      respond_to do |format|
        format.turbo_stream # Para atualizações instantâneas
        format.html { redirect_to tasks_path, notice: "Tarefa criada" }
      end
    else
      render :new, status: :unprocessable_entity
    end
  end
end
```

### Sessões e Autenticação

**Não precisa de JWT ou tokens complexos**. O Hotwire Native usa cookies de sessão normais do Rails:

```ruby path=null start=null
# No Rails, funciona igual a web tradicional
class ApplicationController < ActionController::Base
  before_action :authenticate_user!
  
  private
  
  def authenticate_user!
    redirect_to login_path unless user_signed_in?
  end
end
```

Os cookies ficam salvos no WebView e funcionam automaticamente entre as telas do app.

### Segurança

**HTTPS é obrigatório em produção**:
- iOS: Configure ATS (App Transport Security)
- Android: Configure Network Security Policy
- Rails: Use `force_ssl = true` em produção

**CSRF Protection funciona normalmente**:
```erb path=null start=null
<%= form_with model: @task do |form| %>
  <%= form.hidden_field :authenticity_token, value: form_authenticity_token %>
  <!-- campos do formulário -->
<% end %>
```

---

## Boas Práticas

### Estrutura de Projeto Rails

**Mantenha controllers enxutos**:
```ruby path=null start=null
class TasksController < ApplicationController
  def index
    @tasks = current_user.tasks.order(:created_at)
  end
  
  def create
    @task = CreateTaskService.call(current_user, task_params)
    
    if @task.persisted?
      respond_to do |format|
        format.turbo_stream
        format.html { redirect_to tasks_path }
      end
    else
      render :new, status: :unprocessable_entity
    end
  end
end
```

### HTML Mínimo e Partials

**Evite duplicação usando partials**:

```erb path=null start=null
<!-- app/views/tasks/index.html.erb -->
<%= turbo_frame_tag "tasks" do %>
  <%= render @tasks %>
<% end %>

<!-- app/views/tasks/_task.html.erb -->
<div id="<%= dom_id(task) %>" class="task">
  <h3><%= task.title %></h3>
  <p>Status: <%= task.done? ? "Feita" : "Pendente" %></p>
  <%= link_to "Editar", edit_task_path(task) %>
</div>
```

### JavaScript Mínimo

**Use Stimulus apenas quando necessário**:

```javascript path=null start=null
// app/javascript/controllers/toggle_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["button", "status"]
  
  connect() {
    // Conecta automaticamente
  }
  
  toggle() {
    // Apenas para casos onde Turbo não é suficiente
    this.buttonTarget.disabled = true
  }
}
```

**HTML com Stimulus**:
```erb path=null start=null
<div data-controller="toggle">
  <button data-action="click->toggle#toggle" data-toggle-target="button">
    Alternar Status
  </button>
  <span data-toggle-target="status"><%= task.status %></span>
</div>
```

### Performance com Turbo Frames

**Use Frames para listas grandes**:
```erb path=null start=null
<!-- Lista principal -->
<%= turbo_frame_tag "task_list" do %>
  <%= render partial: "task", collection: @tasks %>
<% end %>

<!-- Formulário em Frame separado -->
<%= turbo_frame_tag "new_task" do %>
  <%= render "form", task: @task %>
<% end %>
```

### Caching HTTP

**Use ETags nos controllers**:
```ruby path=null start=null
class TasksController < ApplicationController
  def index
    @tasks = current_user.tasks.includes(:user)
    
    if stale?(etag: @tasks, last_modified: @tasks.maximum(:updated_at))
      # Renderiza apenas se mudou
    end
  end
end
```

---

## Exemplos Reais

### Autenticação com Turbo

**Controller de sessões**:
```ruby path=null start=null
class SessionsController < ApplicationController
  def create
    user = User.find_by(email: params[:email])
    
    if user&.authenticate(params[:password])
      session[:user_id] = user.id
      redirect_to root_path, notice: "Bem-vindo!"
    else
      @error = "Email ou senha incorretos"
      render :new, status: :unprocessable_entity
    end
  end
end
```

**Formulário de login**:
```erb path=null start=null
<%= form_with url: login_path, local: false do |form| %>
  <%= turbo_frame_tag "login_errors" do %>
    <% if @error %>
      <div class="error"><%= @error %></div>
    <% end %>
  <% end %>
  
  <%= form.email_field :email, required: true %>
  <%= form.password_field :password, required: true %>
  <%= form.submit "Entrar" %>
<% end %>
```

### CRUD com Turbo Streams

**Criar tarefa instantaneamente**:
```erb path=null start=null
<!-- app/views/tasks/create.turbo_stream.erb -->
<%= turbo_stream.prepend "tasks", partial: "task", locals: { task: @task } %>
<%= turbo_stream.replace "new_task_form", partial: "form", locals: { task: Task.new } %>
```

**Atualizar status**:
```erb path=null start=null
<!-- app/views/tasks/toggle.turbo_stream.erb -->
<%= turbo_stream.replace dom_id(@task), partial: "task", locals: { task: @task } %>
```

### Broadcast em Tempo Real

**No model**:
```ruby path=null start=null
class Task < ApplicationRecord
  belongs_to :user
  
  after_create_commit { broadcast_prepend_to "tasks" }
  after_update_commit { broadcast_replace_to "tasks" }
  after_destroy_commit { broadcast_remove_to "tasks" }
end
```

**Na view**:
```erb path=null start=null
<%= turbo_stream_from "tasks" %>
<%= turbo_frame_tag "tasks" do %>
  <%= render @tasks %>
<% end %>
```

---

## Arquitetura do App

### Estrutura Rails + Hotwire

```text
app/
  controllers/
    application_controller.rb
    tasks_controller.rb
  models/
    task.rb
    user.rb
  views/
    tasks/
      index.html.erb
      _task.html.erb
      create.turbo_stream.erb
  javascript/
    controllers/
      application.js (Stimulus setup)
      task_controller.js (opcional)
```

### Configuração Importmap

Exemplo baseado no seu projeto:

```ruby path=/Users/davidsilveira/Projects/bello_rango/config/importmap.rb start=1
pin "application"
pin "@hotwired/turbo-rails", to: "turbo.min.js"
pin "@hotwired/stimulus", to: "stimulus.min.js"
pin "@hotwired/stimulus-loading", to: "stimulus-loading.js"
pin_all_from "app/javascript/controllers", under: "controllers"
```

### Setup do Stimulus

Configuração padrão do seu projeto:

```javascript path=/Users/davidsilveira/Projects/bello_rango/app/javascript/controllers/application.js start=1
import { Application } from "@hotwired/stimulus"

const application = Application.start()

// Configure Stimulus development experience
application.debug = false
window.Stimulus   = application

export { application }
```

### Arquitetura Native

**iOS Structure**:
```swift path=null start=null
// Session controla a navegação
class MainSessionDelegate: SessionDelegate {
    func session(_ session: Session, didProposeVisit proposal: VisitProposal) {
        // Decide se abre modal ou faz push
        if proposal.url.path.contains("/modal") {
            presentModally(proposal.url)
        } else {
            visit(proposal.url)
        }
    }
}
```

**Android Structure**:
```kotlin path=null start=null
// Navigator controla fragments/activities
class MainNavigator(private val activity: AppCompatActivity) : TurboNavigator {
    override fun navigate(proposal: VisitProposal) {
        when {
            proposal.url.path.contains("/modal") -> {
                // Apresentar modal
            }
            else -> {
                // Navegação normal
            }
        }
    }
}
```

### Gerenciamento de Estado

**Servidor como fonte da verdade**:
- Estado persistente vive no Rails
- Apps nativos apenas exibem e enviam dados
- Cache local mínimo (apenas para UX)

**Estado local apenas para UX**:
```javascript path=null start=null
// Apenas para feedback visual temporário
export default class extends Controller {
  loading() {
    this.element.classList.add("loading")
  }
  
  loaded() {
    this.element.classList.remove("loading")
  }
}
```

---

## Integração Web-Nativo (Bridge)

### Quando Usar Bridge

**Evite sempre que possível**. Use apenas para:
- Acesso à câmera/galeria
- Notificações push
- Funcionalidades específicas do OS

### Exemplo iOS Bridge

```swift path=null start=null
// Registrar handler
session.webView.configuration.userContentController
    .add(self, name: "bridge")

// Receber mensagens do JS
func userContentController(_ userContentController: WKUserContentController, 
                          didReceive message: WKScriptMessage) {
    if let body = message.body as? [String: Any],
       let action = body["action"] as? String {
        switch action {
        case "openCamera":
            presentImagePicker()
        default:
            break
        }
    }
}
```

### Exemplo Android Bridge

```kotlin path=null start=null
// No WebView
webView.addJavascriptInterface(Bridge(), "bridge")

class Bridge {
    @JavascriptInterface
    fun openCamera() {
        // Abrir câmera
    }
}
```

### JavaScript Bridge (Use apenas quando necessário)

```javascript path=null start=null
// Exemplo mínimo - evite usar se possível
function openCamera() {
    if (window.webkit?.messageHandlers?.bridge) {
        window.webkit.messageHandlers.bridge.postMessage({
            action: "openCamera"
        })
    } else if (window.bridge) {
        window.bridge.openCamera()
    }
}
```

---

## Recursos da Comunidade

### Documentação Oficial
- **Hotwire**: https://hotwired.dev
- **Turbo Handbook**: https://turbo.hotwired.dev/handbook
- **Stimulus**: https://stimulus.hotwired.dev
- **Turbo iOS**: https://github.com/hotwired/turbo-ios
- **Turbo Android**: https://github.com/hotwired/turbo-android

### Ferramentas Úteis
- **strong_migrations**: https://github.com/ankane/strong_migrations
- **ViewComponent**: https://viewcomponent.org
- **Importmap Rails**: Rails 7+ default
- **Redis**: Para Action Cable em produção

### Exemplos e Recursos
- **HotwiRe-iOS Demo**: Repositório oficial com exemplos
- **Turbo Native Directory**: Lista de apps usando Turbo Native
- **GoRails**: Tutoriais em vídeo sobre Hotwire

### Onde Pedir Ajuda
- **GitHub Discussions**: Nos repositórios turbo-ios/turbo-android
- **Discord**: Hotwire/StimulusReflex community
- **Stack Overflow**: Tag `hotwire` ou `turbo-rails`

---

## Checklist para Publicação

### Backend (Rails)
- [ ] HTTPS configurado (`force_ssl = true`)
- [ ] CSRF protection ativo
- [ ] Sessions configuradas corretamente
- [ ] Action Cable com Redis em produção
- [ ] Assets precompilados
- [ ] Variáveis de ambiente seguras
- [ ] Logs e monitoramento

### iOS App
- [ ] ATS configurado (apenas HTTPS em prod)
- [ ] Info.plist com permissões necessárias
- [ ] Bundle identifier único
- [ ] Assinatura de código válida
- [ ] Ícones e screenshots
- [ ] Teste no TestFlight
- [ ] Review guidelines da Apple

### Android App
- [ ] Network Security Config (HTTPS obrigatório)
- [ ] Permissions no manifest
- [ ] Package name único
- [ ] APK assinado para release
- [ ] Play Console configurado
- [ ] Teste interno/fechado
- [ ] Políticas do Google Play

### Testes Gerais
- [ ] CRUD funciona via Turbo no navegador
- [ ] CRUD funciona nos apps iOS/Android
- [ ] Autenticação persiste entre sessões
- [ ] Navegação nativa funciona
- [ ] Performance aceitável
- [ ] Funciona offline básico (cache HTTP)

Este manual fornece a base para criar e publicar apps com Hotwire Native. Lembre-se: comece simples, teste constantemente e publique incrementalmente.