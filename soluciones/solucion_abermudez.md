## Ejercicio Semanal -- Alejandro Bermúdez


### Diagrama del flujo de trabajo

![imagen](https://github.com/user-attachments/assets/aa0d692e-2d4a-4b4c-a95d-8d6eebc36cf9)


### Archivos


#### Workflow_principal.yml

```yml

name: Workflow principal para llamar a los reusables 

on:
  push:
    branches:
      - main
      - development 
jobs:
  setear_entorno:
    runs-on: ubuntu-latest
    outputs:
      entorno: ${{ steps.entorno.outputs.entorno  }}
    steps:
          - name: Settear entorno
            id: entorno
            run: |
              if [ "${{ github.ref }}" == "refs/heads/main" ]; then
                 echo "Entorno configurado en Prod"
                 echo "entorno=prod" >> $GITHUB_OUTPUT
              elif [ "${{ github.ref }}" == "refs/heads/development" ]; then
                echo "Entorno configurado en UAT"
                echo "entorno=uat" >> $GITHUB_OUTPUT
              else
                echo "Rama distinta a main / develop, entorno no configurado"
              fi
  
  lanzar_CI:
      needs: setear_entorno
      uses: ./.github/workflows/CI_workflow.yml
      secrets: inherit
      with:
        entorno: ${{ needs.setear_entorno.outputs.entorno  }}

          
  lanzar_CD:
      needs: 
        - lanzar_CI
        - setear_entorno
      uses: ./.github/workflows/CD_workflow.yml
      secrets: inherit
      with:
        nombre_completo: ${{ needs.lanzar_CI.outputs.nombre_completo}}
        entorno: ${{ needs.setear_entorno.outputs.entorno  }}

            
            

```


#### CI_worfklow.yml

```yml

  name: CI workflow

on:
  workflow_call:
    inputs:
      entorno:
        required: true
        type: string
        
    outputs:
      nombre_completo:
        value: ${{jobs.build.outputs.nombre_completo}}

jobs:



  testear:
      runs-on: ubuntu-latest
      if: ${{ inputs.entorno  == 'prod' }}
      steps:
          - name: test
            run: | 
              echo "Haciendo tests de cobertura de código..."
              sleep 5

  build:
    if: ${{ always() }}
    needs: testear
    runs-on: ubuntu-latest
    outputs:
      nombre_completo: ${{steps.tag-imagen.outputs.nombre_completo}}
    environment: ${{inputs.entorno}}
    steps:
    - name: Clonar repo
      uses: actions/checkout@v3
      

  
    - name: Leer la versión del package.json
      id: coger_version
      run: |
           version=$(jq -r .version package.json)
           echo "version=$version" >> $GITHUB_OUTPUT
      shell: bash

    - name: Taggear mediante custom action
      id: tag-imagen
      uses: ./.github/actions/  
      with:
        nombre_imagen: 'angular_ab'
        version: "${{ steps.coger_version.outputs.version }}"

    - name: Construir imagen
      run: docker build -t ${{ vars.USUARIO_DOCKER }}/${{steps.tag-imagen.outputs.nombre_completo}} .




    
    - name: Hacer login en DockerHub
      uses: docker/login-action@v3
      with:
        username: ${{ vars.USUARIO_DOCKER }}
        password: ${{ secrets.PASSWORD_DOCKER }}

    - name: Verificar nombre completo de la imagen
      run: echo "Nombre completo de la imagen --> ${{ steps.tag-imagen.outputs.nombre_completo }}"


    - name: Subir imagen al registry
      run: docker push ${{ vars.USUARIO_DOCKER }}/${{steps.tag-imagen.outputs.nombre_completo}}

```
