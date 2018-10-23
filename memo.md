# RubyOnRails notes

## Prerequisites

installer `rbenv` qui est un gestionnaire de version pour ruby
intallera directement la dependence a `ruby-build`

## Setup a new project

utiliser `rails new [name]` cest un equivalent de crapp
pour lancer le serveur de dev dans le repertoire courant: `rails server`

## Troubleshooting

Sur High sierra, difficultes rencontrees lors de la creation dun nouveau projet ruby notament lors de linstallation dune gem
`sudo gem install sqlite3 -- --with-sqlite3-lib=/usr/lib`
