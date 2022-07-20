# docker_compose("./docker-compose.yaml")

# SETUP
k8s_yaml([
    'kube/common/pv.yaml',
    'kube/common/pvc.yaml',
    'kube/common/app-config-development.yaml',
    'kube/common/app-secret.yaml',
    'kube/database/secret.yaml',
    'kube/redis/pvc.yaml',
    'kube/database/pvc.yaml',
])
k8s_resource(
    objects=[
        'laravel-in-kubernetes-volume:persistentvolume',
        'laravel-in-kubernetes-pvc:persistentvolumeclaim',
        'laravel-in-kubernetes:configmap',
        'laravel-in-kubernetes:sealedsecret',
        'laravel-in-kubernetes-mysql:sealedsecret',
        'laravel-in-kubernetes-mysql-pvc:persistentvolumeclaim',
        'laravel-in-kubernetes-redis-pvc:persistentvolumeclaim'
    ],
    new_name='cluster',
    labels=['setup']
)

# DATABASE
k8s_yaml([
    'kube/database/statefulset.yaml',
    'kube/database/service.yaml',
])
k8s_resource(
    'laravel-in-kubernetes-mysql',
    new_name='mysql',
    resource_deps=['cluster'],
    labels=['database']
)

# REDIS
k8s_yaml([
    'kube/redis/statefulset.yaml',
    'kube/redis/service.yaml'
])
k8s_resource(
    'laravel-in-kubernetes-redis',
    new_name='redis',
    resource_deps=['cluster'],
    labels=['database']
)

# WEB
docker_build('stickeepaul/lik-web_server', '.',
    target='web_server',
    live_update=[
        sync('.', '/opt/apps/laravel-in-kubernetes'),
    ],
)
k8s_yaml([
    'kube/webserver/deployment-dev.yaml',
    'kube/webserver/service.yaml',
    'kube/webserver/ingress.yaml'
])
k8s_resource(
    'laravel-in-kubernetes-webserver',
    new_name='web',
    resource_deps=['php'],
    port_forwards=[port_forward(8080, 80, name='web')], # don't do this in "production" dev, use ingress instead
    labels=['app'],
)
local_resource(
    'sync public files',
    cmd='touch public/index.php',
    resource_deps=['web'],
    labels=['setup']
)

# FPM
docker_build('stickeepaul/lik-fpm_server', '.',
    target='fpm_server',
    live_update=[
        sync('.', '/opt/apps/laravel-in-kubernetes')
    ],
    ignore=['kube']
)
k8s_yaml(listdir('kube/fpm'))
k8s_resource(
    'laravel-in-kubernetes-fpm',
    new_name='php',
    resource_deps=['mysql', 'redis'],
    labels=['app']
)
php_route_clear_fix = '''
set -eu
POD_NAME="$(tilt get kubernetesdiscovery "php" -ojsonpath='{.status.pods[0].name}')"
kubectl exec "$POD_NAME" -c fpm -- php artisan route:clear
'''
local_resource(
    'php route:clear (fix)',
    cmd=php_route_clear_fix,
    resource_deps=['php'],
    labels=['setup']
)

# # CRON
# docker_build('stickeepaul/lik-cli', '.',
#     target='cron',
#     live_update=[
#         sync('.', '/opt/apps/laravel-in-kubernetes')
#     ],
#     # ignore=['kube', 'resources']
# )
# k8s_yaml(listdir('kube/cron'))
# k8s_resource(
#     'laravel-in-kubernetes-scheduler',
#     new_name='cron',
#     resource_deps=['mysql', 'redis'],
#     labels=['app']
# )

# FRONTEND
docker_build('stickeepaul/lik-frontend', '.',
    target='frontend',
    live_update=[
        sync('./', '/opt/apps/laravel-in-kubernetes/'),
    ],
    entrypoint=['sleep', '10000'],
)
k8s_yaml('kube/frontend/deployment-dev.yaml')
k8s_resource(
    'laravel-in-kubernetes-frontend',
    new_name='frontend',
    labels=['app'],
    resource_deps=['php'],
)


load('ext://uibutton', 'cmd_button', 'text_input', 'location')

pod_exec_script = '''
set -eu
POD_NAME="$(tilt get kubernetesdiscovery "php" -ojsonpath='{.status.pods[0].name}')"
kubectl exec "$POD_NAME" -c fpm -- php artisan make:$command
kubectl cp "$POD_NAME":/opt/apps/laravel-in-kubernetes/app app -c fpm
'''
cmd_button('php_artisan_make',
    argv=['sh', '-c', pod_exec_script],
    resource='php',
    # location=location.NAV,
    icon_name='',
    text='artisan make',
    inputs=[
        text_input('command'),
    ]
)

# doesn't work because there's no composer in the php container
pod_exec_script = '''
set -eu
POD_NAME="$(tilt get kubernetesdiscovery "php" -ojsonpath='{.status.pods[0].name}')"
kubectl exec "$POD_NAME" -c fpm -- composer $package
kubectl cp "$POD_NAME":/opt/apps/laravel-in-kubernetes/composer.json composer.json -c fpm
kubectl cp "$POD_NAME":/opt/apps/laravel-in-kubernetes/composer.lock composer.lock -c fpm
'''
cmd_button('php_composer_require',
        argv=['sh', '-c', pod_exec_script],
        resource='php',
        # location=location.NAV,
        icon_name='',
        text='composer',
        inputs=[
            text_input('package'),
        ]
)

pod_exec_script = '''
set -eu
POD_NAME="$(tilt get kubernetesdiscovery "frontend" -ojsonpath='{.status.pods[0].name}')"
kubectl exec "$POD_NAME" -- npm $command && \
kubectl cp "$POD_NAME":/opt/apps/laravel-in-kubernetes/package.json package.json
kubectl cp "$POD_NAME":/opt/apps/laravel-in-kubernetes/package.lock package.lock
'''
cmd_button('frontend_npm_install',
        argv=['sh', '-c', pod_exec_script],
        resource='frontend',
        # location=location.NAV,
        icon_name='',
        text='npm',
        inputs=[
            text_input('command'),
        ]
)
