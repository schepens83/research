# Notes

- Started investigation into how `ActionController::Base::MODULES` is loaded into `ActionController::Base`.
- Workspace has existing Rails architecture pattern notes, but no Rails source checkout. Need inspect upstream Rails source or installed gem source.
- Found installed Rails source at `actionpack-8.1.3`.
- In `lib/action_controller.rb`, `ActionController` extends `ActiveSupport::Autoload` and declares autoloads for `Base`, `Metal`, and the `metal/*` modules.
- In `lib/action_controller/base.rb`, `Base < Metal` defines `without_modules`, then `MODULES`, then explicitly repeats that ordered list as `include ...` statements.
- Verified with Ruby that `ActionController::Base::MODULES == ActionController::Base.without_modules`.
- Verified `ActionController::Base.ancestors`: modules included later appear earlier in the method lookup chain, e.g. `ParamsWrapper`, `Instrumentation`, `Rescue`, then earlier includes.
- Contrast: `ActionController::API` defines a similar `MODULES` constant and then uses `MODULES.each { |mod| include mod }`; `ActionController::Base` currently does not do that loop.
