# docker_compose("./docker-compose.yaml")

# SETUP
k8s_yaml([
    'kube/common/pv.yaml',
    'kube/common/app-config-development.yaml',
    'kube/common/app-secret.yaml',
    'kube/database/secret.yaml',
    'kube/redis/pvc.yaml',
    'kube/database/pvc.yaml',
])
k8s_resource(
    objects=[
        'laravel-in-kubernetes-volume:persistentvolume',
        'laravel-in-kubernetes:configmap',
        'laravel-in-kubernetes:sealedsecret',
        'laravel-in-kubernetes-mysql:sealedsecret',
        'laravel-in-kubernetes-mysql-pvc',
        'laravel-in-kubernetes-redis-pvc'
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
        sync('.', '/opt/apps/laravel-in-kubernetes')
        # sync('./public', '/opt/apps/laravel-in-kubernetes/public')
    ],
)
k8s_yaml(listdir('kube/webserver'))
k8s_resource(
    'laravel-in-kubernetes-webserver',
    new_name='web',
    resource_deps=['php'],
    port_forwards=[port_forward(8080, 80, name='web')],
    labels=['app'],
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
        sync('./resources', '/opt/apps/laravel-in-kubernetes/resources')
    ],
    entrypoint=['npm', 'run', 'watch'],
)
k8s_yaml(listdir('kube/frontend'))
k8s_resource('laravel-in-kubernetes-frontend', new_name='frontend', labels=['app'], resource_deps=['php'])
