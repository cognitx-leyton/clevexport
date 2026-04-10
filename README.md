# Clevexport

***Async Excel export for large Laravel datasets -- chunked, queued, and crash-free.***

![License](https://img.shields.io/github/license/cognitx-leyton/clevexport?style=flat-square)
![Stars](https://img.shields.io/github/stars/cognitx-leyton/clevexport?style=flat-square)
![PHP](https://img.shields.io/badge/php-%3E%3D7.4-8892BF?style=flat-square&logo=php&logoColor=white)
![Laravel](https://img.shields.io/badge/laravel-6.x--8.x-FF2D20?style=flat-square&logo=laravel&logoColor=white)

---

## Table of Contents

- [Motivation](#motivation)
- [Installation](#installation)
- [Usage](#usage)
- [Exportable](#exportable)
- [Transformer](#transformer)

---

## Motivation
The goal of this Laravel package is to execute the exportation to **EXCEL** large data/records that may cause in the crash of the server or a timeout. The idea is to divide the process in sub operations managable and easy to perform.

## Installation
````
compose require leyton/clevexport
````
After the installation make sure to publish the assets
````
php artisan vendor:publish --provider="Leyton\ClevExportServiceProvider"
````

You will find the ````config/clevexport.php```` file containing all the configurations needed.
````php
return [
    // array of listeners that will be executed once the execution is done
    'listeners' => [], 
    
    // if you want to stock the user who started the export
    'with_owner' => true,
    
    // the guard    
    'guard' => 'web',
    
    // the foreign key name in the exports table
    'owner_id' => 'user_id',
    
    // The Authenticable class
    'owner_class' => \App\Models\User:class,
    
    // Number of chunks
    'chunks' => 10,
];
````


Then you can run your migration

````
php artisan migrate
````

## Usage

The ``QueryFinder`` should be provided with an instance of an object that implements the ``IseExportable`` Interface

````php

 $dossierExporter = QueryFinder::getInstance($this->defaultDossierService, $this->transformer);

 PreparingExportJob::dispatch($dossierExporter, $request->all())->delay(now()->addSecond());
````

A second Parameter is optional and if is provided it should implement the ``ShouldHandleResult`` It is where you can perform extra work on the results and provide the headers in a ``ExportTransformed`` container
If the second parameter is not provided then the headers will be the column names selected from the query.

## Exportable
````php

class UserExportable implements IsExportable
{

    /**
     * @param array $params
     * @return Builder
     */
    public function query(array $params): Builder
    {
        return  User::select('id', 'name', 'email')
            ->addSelect([
                'title' => Post::selectRaw('SUBSTRING(`content`, 1, 10) as `title`')->limit(1)
            ]);
    }
}
````

## Transformer

````php 
class UserTransformer implements ShouldHandleResult
{

    public function transform($data): ExportTransformed
    {
        $data =   $data->map(function(User $user){
            $user->most_commented = optional($user->posts()->withCount('comments')->first())->comments_count;
            $user->comments_count = $user->comments()->count();
            $user->posts_count = $user->posts()->count();
            return $user;
        });

        return  new ExportTransformed(['id', 'name', 'email', 'title', 'Most commented', 'Comments count', 'Posts count'], $data->toArray());
    }
}
````

---

## License

This project is licensed under the Apache 2.0 License - see the [LICENSE](LICENSE) file for details.

---

<p align="center">
  <sub>Built with care by <a href="https://cognitx.leyton.com/">Leyton CognitX</a></sub>
</p>
