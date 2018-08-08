Rut Chileno
===========

Esta librería implementa una clase Rut como un *value object* inmutable, incluyendo
una api de validación flexible y extendible. 

Además, posee un validador para `symfony/validator`, un *form type* para `symfony/form`
y un *type* para `doctrine/dbal`. 

Sólo es compatible con PHP 7.1 o superior.

## Uso
Simplemente instancia una nueva clase con un rut en cualquier formato:

```php
<?php

use MNC\ChileanRut\Rut;

$rut = new Rut('23.546.565-4');

// Si prefieres, puedes usar el factory method

$rut = Rut::fromString('23546565-4');
```

Por defecto, la clase Rut no se valida a sí misma, lo que puede significar que
el objeto sea instanciado en un estado inconsistente. Si esto no te gusta, puedes pasarle
el `SimpleRutValidator` al constructor, y el Rut será validado al momento de ser instanciado.

```php
<?php

use MNC\ChileanRut\Rut;
use MNC\ChileanRut\Validator\SimpleRutValidator;

$rut = new Rut('23.546.565-4', new SimpleRutValidator());

// Esto lanzará un InvalidRutException si el rut no es valido.
```

### Validación Personalizada de Rut
El `SimpleRutValidator` no es más que la implementación del validador clásico de Rut,
el algoritmo de módulo 11. Esto verifica que un Rut es algoritmicamente correcto, pero 
no valida que es real.

Por ello, proveemos la interfaz `RutValidator`. Con ella, puedes crear tus propias
reglas de validación, como llamar a un web service o consultar una base de datos
para verificar si un Rut es real o no. Te recomiendo mirar la interfaz para
implementarla correctamente.

De todas formas, aquí hay un ejemplo que va a buscar un rut a un web service.

```php
<?php

use MNC\ChileanRut\Validator\RutValidator;
use MNC\ChileanRut\Rut;
use MNC\ChileanRut\Exception\InvalidRutException;
use App\Rut\WebServiceRutChecker;

class MyCustomRutValidator implements RutValidator
{
    private $rutChecker;
    
    /**
     * MyCustomRutValidator constructor.
     * @param WebServiceRutChecker $rutChecker
     */
    public function __construct(WebServiceRutChecker $rutChecker) 
    {
        $this->rutChecker = $rutChecker;
    }
    
    /**
     * @param Rut $rut
     */
    public function validate(Rut $rut) : void
    {
        // Por debajo, esta clase ficticia haría una llamada a un web service preguntando
        // si el Rut existe.
        if ($this->rutChecker->doesRutExist($rut->format())) {
            return;
        }
        throw new InvalidRutException($rut, 'This rut does not exist');
    }
}

```

> NOTA: La implementación de cualquier validador DEBE arrojar un InvalidRutException cuando
el Rut no es válido. De lo contrario, el Rut se toma como válido.

### Usando múltiples validadores
Proveemos un `ChainRutValidator` que puedes usar para validar un rut contra múltiples
validadores. Esto permite ejecutar cadenas de validación, como ver primero si un rut es
válido algorítmicamente antes de verificarlo contra un web service.

Usarlo es simple:

```php
<?php

use MNC\ChileanRut\Rut;
use MNC\ChileanRut\Validator\ChainRutValidator;
use MNC\ChileanRut\Validator\SimpleRutValidator;
use App\Rut\DatabaseRutValidator;

$chainValidator = new ChainRutValidator();
$chainValidator
    ->append(new SimpleRutValidator())
    ->append(new DatabaseRutValidator());

$rut = new Rut('14.245.245-2');

$chainValidator->validate($rut);
```

### Formateando Ruts a String

Una vez creado el objeto Rut, puedes formatearlo a string en el formato que tu quieras.
Esto se hace a través del método format y cómo parámetro acepta el valor
de una de las constantes FORMAT_ de la clase Rut.

```php
<?php

use MNC\ChileanRut\Rut;

$rut = new Rut('34244223-4');

echo $rut->format(Rut::FORMAT_CLEAR);       // Va a imprimir 342442234
echo $rut->format(Rut::FORMAT_READABLE);    // Va a imprimir 34.244.223-4
echo $rut->format(Rut::FORMAT_HYPHENED);    // Va a imprimir 34244223-4
echo $rut->format(Rut::FORMAT_HIDDEN);      // Va a imprimir 34.***.***-4
```

### Utilidades
Esta librería provee una clase llamada `CorrelativeUtils` que tiene algunas utilidades
interesantes. Posee tres métodos:

```php
<?php

use MNC\ChileanRut\Util\CorrelativeUtils;

// Este método devuelve el digito verificador de un correlativo.
CorrelativeUtils::findVerifierDigit('34525252');

// Este método devuelve una instancia de Rut válida, sólo con el correlativo.
CorrelativeUtils::createValidRutOnlyFromCorrelative('34525252');

// Este método devuelve instancia de Rut autogenerada algoritmicamente válida.
CorrelativeUtils::autoGenerateValidRut();
```

## Integraciones con Liberías de Terceros

### Doctrine DBAL
Esta libería provee un custom type para doctrine llamado `RutType`. Puedes registrarla
en el Dbal para usarla en tus mappings de doctrine y automáticamente mappear tu
el valor de tu db a un objeto rut.

### Symfony Validator
Además, esta libería cuenta con un validador para Symfony Validator, que te 
permite beneficiarte de las anotaciones del componente de validación de Symfony.
Como dependencia opcional necesita una instancia de `RutValidator`. Si ninguna es proveída,
se utiliza el `SimpleRutValidator`. Solo puedes usar el validador contra una instancia de `Rut`.

### Symfony Form Type
Por último, esta libería cuenta con un Symfony Form Type que puedes añadir en tus
formularios HTML, para que puedas autoinstanciar la clase y poner lógica de 
validación en ella sin problema, y añadirla a tus otros tipos.

## FAQ

### ¿Cómo nació y por qué esta librería?
Esta libería nace de la necesidad de estandarizar una clase Rut común para todos mis proyectos
PHP.
Si bien es cierto, hay muchas liberías con implementaciones de Rut chilenos en PHP,
muchas de ellas tienen notorias deficiencias:

1. No están testeadas unitariamente,
2. No separan bien responsabilidades, como la lógica de validación con la de instanciación.
3. No proveen validación extensible por medio de interfaces, limitando la validación
solo a ser algorítmica.
4. Están acopladas a un framework
5. No proveen herramientas ni integraciones con librerías de terceros.

### ¿Por qué PHP 7.1?
El fin del soporte de PHP 5.6 será a fines de 2018. PHP 7.1 es una de las últimas
versiones estables, y me beneficio mucho de su sistema de tipado estricto en esta libería.
