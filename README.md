# GitHub CLI y GitHub Actions

## Autenticación

```bash
echo "{{ GH_TOKEN }}" | gh auth login --with-token
```

Aquí, el token es enviado por ``echo`` a través de una tubería (|), lo que significa que el token se pasa como entrada estándar (stdin) al comando, como si el usuario lo escribiera después de ejecutar el comando.